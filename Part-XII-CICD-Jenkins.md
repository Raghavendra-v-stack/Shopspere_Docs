# The ShopSphere DevOps Book
## Part XII — CI/CD with Jenkins

---

### Where we left off

ShopSphere has a working, secure, autoscaling Kubernetes deployment, packaged as a Helm chart. Every deploy so far has been manual — you, typing `helm upgrade` or `kubectl apply` yourself. This Part automates that entire path, from a developer pushing code to a tested, deployed release, with no manual steps in between.

---

## Chapter 11.1 — Why CI/CD, Precisely

**Continuous Integration (CI)** means every code change is automatically built and tested as soon as it's pushed — catching problems within minutes, not days later during a manual "integration" phase. **Continuous Delivery/Deployment (CD)** extends this: a change that passes CI is automatically packaged and (delivery) made ready to deploy, or (deployment) actually deployed, with no manual steps.

**Why ShopSphere specifically needs this now.** Recall Chapter 5.1's Problem 3: deploys are risky and manual. Manually running `docker build`, `docker push`, and `helm upgrade` correctly, every time, for every change, doesn't scale past one or two engineers, and is exactly the kind of repetitive, error-prone human process this book has worked to eliminate at every previous layer — image building (Docker), orchestration (Kubernetes), configuration templating (Helm). CI/CD removes it from the deployment path entirely.

### The full pipeline we're building

```text
Developer
   |
   v
Git  (push to a branch, or merge to main)
   |
   v
Jenkins  (triggered by a webhook)
   |
   +--> Checkout code
   |
   +--> Run tests
   |
   +--> Docker build
   |
   +--> Security scan  (recall Part V — this gate belongs right here)
   |
   +--> Push image --> ECR
   |
   +--> Deploy --> Kubernetes (via Helm)
   |
   +--> Smoke test
   |
   v
Done — new version live, or pipeline fails and nothing changed
```

The core principle behind every stage: **fail fast, and fail before anything reaches production.** Each stage is a gate — if tests fail, the pipeline stops before a broken image is even built; if the security scan finds a critical vulnerability, the pipeline stops before that image is ever pushed to ECR at all.

---

## Chapter 11.2 — Jenkins Fundamentals

### Jenkins architecture: controller and agents

**Simple explanation:** the Jenkins controller is the brain that manages pipelines and the web UI; the actual work (checking out code, running builds) happens on separate machines or containers called agents, which the controller dispatches jobs to.

**Proper definition:** the **Jenkins controller** schedules jobs, serves the UI, and stores configuration and build history; **Jenkins agents** are the workers that actually execute pipeline steps, connecting back to the controller.

**Why this split matters, connecting back to Part V's Docker socket warning.** A common, practical setup runs Jenkins agents themselves as Kubernetes Pods (via the Jenkins Kubernetes plugin), spun up on demand for each build and torn down afterward — clean, isolated, and elastic. This is precisely the scenario Chapter 4.1 flagged: an agent that needs to build Docker images inside a container needs *some* way to actually build them, and the naive approach (mounting the host's Docker socket into the agent Pod) reintroduces exactly the host-takeover risk described there. We'll address this properly in Chapter 11.4.

### Pipeline as code: the Jenkinsfile

**Simple explanation:** instead of clicking through a web UI to configure a build job, you write the entire pipeline as a file, checked into the same Git repository as the application code — versioned, reviewable, and reproducible, exactly like everything else we've built in this book.

**Proper definition:** a **Jenkinsfile** defines a Jenkins pipeline using a Groovy-based DSL, typically in **Declarative Pipeline** syntax (the more structured, readable, and now-standard form) rather than the older, more flexible-but-verbose Scripted Pipeline syntax.

### Credentials

Jenkins has a dedicated **Credentials** store, letting pipelines reference secrets (registry passwords, kubeconfig files, API tokens) by an ID, without ever hardcoding the actual secret value into the Jenkinsfile itself, where it would sit in plaintext in version control — exactly the same principle from Part III/V's "never bake a secret into an image or a Dockerfile" guidance, applied here to pipeline configuration instead.

### Webhooks

A **webhook** is how Git actually triggers Jenkins automatically: when code is pushed, the Git hosting provider (GitHub, GitLab, etc.) sends an HTTP request to a Jenkins endpoint, which starts a new pipeline run — this is the literal mechanism behind the "Git → Jenkins" arrow at the top of this chapter's pipeline diagram, replacing a human needing to manually click "build."

---

## Chapter 11.3 — Building ShopSphere's Jenkinsfile, Stage by Stage

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                    - name: builder
                      image: gcr.io/kaniko-project/executor:debug
                      command: ["sleep"]
                      args: ["9999999"]
            '''
        }
    }

    environment {
        ECR_REPO = "123456789012.dkr.ecr.us-east-1.amazonaws.com/shopsphere-backend"
        IMAGE_TAG = "${env.GIT_COMMIT.take(7)}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                sh 'pip install -r requirements.txt --break-system-packages'
                sh 'pytest tests/ --junitxml=results.xml'
            }
            post {
                always {
                    junit 'results.xml'
                }
            }
        }

        stage('Build Image') {
            steps {
                container('builder') {
                    sh """
                        /kaniko/executor \
                          --context=`pwd` \
                          --destination=${ECR_REPO}:${IMAGE_TAG} \
                          --no-push
                    """
                }
            }
        }

        stage('Security Scan') {
            steps {
                sh "trivy image --exit-code 1 --severity CRITICAL ${ECR_REPO}:${IMAGE_TAG}"
            }
        }

        stage('Push to ECR') {
            steps {
                container('builder') {
                    sh """
                        /kaniko/executor \
                          --context=`pwd` \
                          --destination=${ECR_REPO}:${IMAGE_TAG} \
                          --destination=${ECR_REPO}:latest
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'shopsphere-kubeconfig', variable: 'KUBECONFIG')]) {
                    sh """
                        helm upgrade --install shopsphere-prod ./chart/shopsphere-backend \
                          --namespace shopsphere \
                          --set image.tag=${IMAGE_TAG} \
                          --wait --timeout 5m
                    """
                }
            }
        }

        stage('Smoke Test') {
            steps {
                sh """
                    curl -sf --retry 5 --retry-delay 5 \
                      https://api.shop.example.com/health
                """
            }
        }
    }

    post {
        failure {
            echo "Pipeline failed — no changes were left partially applied to production."
        }
        success {
            echo "Deployed ${IMAGE_TAG} to production successfully."
        }
    }
}
```

Let's walk through the choices in this Jenkinsfile deliberately, because several of them are directly resolving problems flagged earlier in the book.

### Checkout

`checkout scm` pulls the exact commit that triggered this pipeline run — nothing new conceptually, just automating the `git pull` a human used to do manually in ShopSphere's original "day one" setup from Chapter 0.1.

### Test

Tests run *before* any image is even built. This ordering is deliberate: there's no point spending time and resources building and scanning an image for code that's already known to be broken — fail as early and as cheaply as possible in the pipeline.

### Build Image — and why this Jenkinsfile uses kaniko, not the Docker socket

This is the direct payoff of Chapter 11.2's Docker-socket foreshadowing. Instead of mounting `/var/run/docker.sock` into the Jenkins agent Pod (Chapter 4.1's dangerous pattern — full host Docker daemon control from inside a build container), this pipeline uses **kaniko**, a tool purpose-built to build container images *inside* a container, from a Dockerfile, **without needing a Docker daemon or elevated/privileged access at all**. This directly solves the exact tradeoff named back in Chapter 4.1 — a real CI build workflow with a properly scoped, non-privileged approach, rather than the convenient-but-dangerous shortcut.

**Interview question (advanced, and directly rewards having read Part V carefully):** "Why might a Kubernetes-based CI pipeline use a tool like kaniko instead of mounting the Docker socket into a build Pod?" — Mounting the Docker socket grants the build Pod effective control over the entire host's Docker daemon — equivalent to root on the node — which is a serious security risk for something as routinely triggered and externally-influenced as a CI build job. kaniko builds images from a Dockerfile entirely in userspace, inside its own container, without requiring a Docker daemon or privileged access, closing off that entire risk.

### Security Scan

This gate runs `trivy` (introduced by name back in Part V) with `--exit-code 1` on any `CRITICAL` finding — meaning the shell command itself returns a nonzero exit code, which Jenkins interprets as a failed stage, **stopping the pipeline before the image is ever pushed anywhere**. This is precisely the automated version of the manual `docker scout cves` step from Part V's lab — now a real, enforced gate instead of a step someone might remember, or might not, to run by hand.

### Push to ECR

Notice this is a genuinely separate stage from the build — the image is built and scanned *before* it's pushed anywhere at all. If the scan stage above fails, this stage never runs, and no vulnerable image ever reaches the registry.

### Deploy to Kubernetes

`helm upgrade --install` is used rather than `helm install` — `--install` makes this idempotent: it installs the release if it doesn't exist yet, or upgrades it if it does, which matters because this same Jenkinsfile stage runs on every single pipeline execution, not just the first one. `--wait --timeout 5m` makes Jenkins actually wait for the rollout to genuinely finish (recall `kubectl rollout status` from Chapter 9.2) rather than reporting success the instant `helm upgrade` returns, before the new Pods have actually become Ready.

The kubeconfig credential is pulled from Jenkins's Credentials store via `withCredentials`, never hardcoded — exactly Chapter 11.2's principle, applied concretely.

### Smoke Test

**What a smoke test is, precisely.** A **smoke test** is a fast, minimal check *after* deployment that the application is genuinely serving real traffic correctly — not a full test suite, just enough to catch "the deploy technically succeeded, but the app is actually broken in production" before declaring victory. Here, it's a simple health-endpoint check against the real, live URL — the actual last line of defense in the entire pipeline.

### Rollback, wired into CI/CD

If the smoke test fails, the `post { failure { ... } }` block runs. A more complete production pipeline would add an explicit automated rollback step here — `helm rollback shopsphere-prod` — rather than just alerting a human, closing the loop all the way back to Chapter 10.4's Helm rollback mechanism, now triggered automatically by a failed smoke test rather than manually by an engineer.

---

## Chapter 11.4 — Jenkins Agents, Properly

We touched this in Chapter 11.2 — let's make the production reasoning explicit, because "how does Jenkins actually build things safely" is a genuinely common practical question.

**Static agents** — long-running Jenkins worker machines, always on, always available. Simple, but wasteful (paying for idle capacity) and prone to configuration drift over time (subtly different tool versions accumulating across builds, unless carefully managed).

**Dynamic/ephemeral agents (what ShopSphere's Jenkinsfile above uses)** — a fresh Pod, spun up specifically for this one pipeline run, using the exact container image(s) declared right in the Jenkinsfile, and destroyed immediately afterward. This directly mirrors the "disposable, replaceable" philosophy from Part III/VI — no configuration drift is even possible, because nothing persists between builds at all; every single build starts from a known-clean, declared state.

---

## Chapter 11.5 — Checkpoint

**Beginner:**
1. What's the difference between Continuous Integration and Continuous Deployment?
2. What does a Jenkinsfile actually define?

**Intermediate:**
3. Why does the Jenkinsfile above run tests *before* building the Docker image, rather than after?
4. What's the difference between "Push to ECR" happening as its own stage, versus happening as part of "Build Image"? Why does that separation matter here specifically?

**Advanced:**
5. Explain, precisely, why kaniko is used instead of mounting the Docker socket, tying your answer back to Part V.
6. What does a smoke test check that unit tests and a security scan don't?

**Scenario:**
7. A pipeline run passes every stage, including the smoke test, but customers report the checkout flow is broken ten minutes later. What does this suggest about the smoke test's coverage, and how would you improve it without turning it into a full, slow test suite?

---

### Hands-On Lab 11.1 — Run ShopSphere's pipeline end to end, locally

**Objective:** Get a real Jenkins instance running a version of this pipeline against your kind cluster and local ECR credentials.

**Prerequisites:** Docker; a kind cluster with the `shopsphere` Helm chart from Part XI available; AWS CLI configured with an ECR repository created (Part V, Chapter 4.3).

**Cost Warning:** this lab uses the same ECR repository pattern as Part V's lab — genuinely low storage cost for a handful of small images, not unconditionally free forever. Clean up afterward, as always.

**Steps:**

1. Run Jenkins locally as a container, with access to the Docker host and your kubeconfig mounted in for this lab's simplified setup (a real production Jenkins would use the Kubernetes-agent-per-build pattern from Chapter 11.4 instead — this lab simplifies to keep the exercise approachable):
   ```bash
   docker run -d --name jenkins \
     -p 8080:8080 -p 50000:50000 \
     -v jenkins_home:/var/jenkins_home \
     jenkins/jenkins:lts
   ```

2. Open `http://localhost:8080`, complete the setup wizard, and install the "Pipeline" and "Kubernetes" plugins.

3. Add credentials in Jenkins's Credentials store: your AWS credentials (for ECR push) and your kind cluster's kubeconfig, matching the `shopsphere-kubeconfig` ID referenced in the Jenkinsfile above.

4. Create a new Pipeline job pointing at a Git repository containing ShopSphere's backend code, the Dockerfile from Part III, the Helm chart from Part XI, and a Jenkinsfile adapted from Chapter 11.3 (simplify the `agent` block to `agent any` for this local lab, since we're not running the full Kubernetes-agent setup here).

5. Trigger a build manually (`Build Now`) and watch each stage execute in the Jenkins UI's pipeline view.

**Expected result:** all stages run in order and go green; the image appears in your ECR repository afterward; `helm list -n shopsphere` shows an updated release.

**Verification:** `kubectl get pods -n shopsphere` shows Pods running the newly-built image tag — confirm with `kubectl get deployment shopsphere-backend -n shopsphere -o jsonpath='{.spec.template.spec.containers[0].image}'`.

**Troubleshooting:** if the "Deploy" stage fails with an auth error, double check the kubeconfig credential was correctly added and correctly referenced by ID — this is far and away the most common setup mistake in this lab.

**Cleanup:**
```bash
docker rm -f jenkins
docker volume rm jenkins_home
aws ecr batch-delete-image --repository-name shopsphere-backend \
  --image-ids imageTag=<the-tag-you-pushed>
```

**Challenge:** intentionally break a test in the application code, push it, and confirm the pipeline correctly stops at the Test stage — proving nothing gets built, scanned, or deployed for genuinely broken code, exactly as the "fail fast" principle from Chapter 11.1 promises.

---

*End of Part XII. Part XIII moves into AWS proper — VPC, IAM, and the networking and identity foundations everything in Part XIV's EKS deployment depends on — followed by Terraform, so that all of ShopSphere's infrastructure, not just its application, becomes version-controlled and repeatable.*
