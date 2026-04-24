### Topic: CI/CD — Level 2

> **Foundation:** [Level 1 — Code Repository](../level-1/code_repository.md) · [Level 1 — Agile Methodology](../level-1/agile_methodology.md)
> **Related:** [Level 2 — Cloud Deployment](./cloud_deployment.md) · [Level 2 — Monitoring and Alerting](./monitoring_and_alerting.md)

**Why it matters (Karat angle)**
Citi uses Jenkins, GitHub Actions, OpenShift, Docker, and Kubernetes. Interviewers ask about your CI/CD pipeline to see if you've shipped code to production — not just committed to main.

**Core concept**

| Phase | What happens | Tools |
|-------|-------------|-------|
| **CI (Continuous Integration)** | Build, test, lint on every commit | Jenkins, GitHub Actions |
| **CD (Continuous Delivery)** | Auto-deploy to staging; manual approval for prod | Jenkins, ArgoCD |
| **CD (Continuous Deployment)** | Auto-deploy to prod (no manual gate) | ArgoCD, Spinnaker |

**Typical pipeline:**
```
Push to feature branch
  → CI: build + unit tests + static analysis (SonarQube)
    → PR review
      → Merge to main
        → CI: build + integration tests + security scan
          → Build Docker image → push to registry
            → Deploy to staging (auto)
              → Smoke tests
                → Deploy to production (manual approval or auto)
```

**GitHub Actions example:**

```yaml
# File: .github/workflows/ci.yml

name: CI Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Build & Test
        run: mvn clean verify

      - name: SonarQube Analysis
        run: mvn sonar:sonar -Dsonar.host.url=${{ secrets.SONAR_URL }}

      - name: Build Docker Image
        if: github.ref == 'refs/heads/main'
        run: docker build -t registry.citi.com/payment-service:${{ github.sha }} .

      - name: Push to Registry
        if: github.ref == 'refs/heads/main'
        run: docker push registry.citi.com/payment-service:${{ github.sha }}
```

**Jenkins pipeline (Jenkinsfile):**

```groovy
// File: Jenkinsfile

pipeline {
    agent any
    environment {
        REGISTRY = 'registry.citi.com'
        IMAGE = 'payment-service'
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }
        stage('Static Analysis') {
            steps {
                sh 'mvn sonar:sonar'
            }
        }
        stage('Docker Build') {
            when { branch 'main' }
            steps {
                sh "docker build -t ${REGISTRY}/${IMAGE}:${env.BUILD_NUMBER} ."
                sh "docker push ${REGISTRY}/${IMAGE}:${env.BUILD_NUMBER}"
            }
        }
        stage('Deploy to Staging') {
            when { branch 'main' }
            steps {
                sh "kubectl set image deployment/${IMAGE} ${IMAGE}=${REGISTRY}/${IMAGE}:${env.BUILD_NUMBER}"
            }
        }
        stage('Deploy to Production') {
            when { branch 'main' }
            input { message 'Deploy to production?' }     // manual approval gate
            steps {
                sh "kubectl --context=prod set image deployment/${IMAGE} ${IMAGE}=${REGISTRY}/${IMAGE}:${env.BUILD_NUMBER}"
            }
        }
    }
}
```

**Docker → OpenShift/Kubernetes workflow:**
```
1. Developer pushes code
2. Jenkins/GHA builds Docker image
3. Image pushed to container registry (Nexus, Docker Hub, ECR)
4. Kubernetes/OpenShift Deployment updated with new image tag
5. Rolling update: new pods start → health checks pass → old pods terminate
6. If new pods fail health checks → rollback automatically
```

**Key concepts:**

| Concept | What it is |
|---------|-----------|
| **Rolling update** | Replace pods one at a time (zero downtime) |
| **Blue-green** | Run old + new in parallel; switch traffic |
| **Canary** | Route 5% of traffic to new version; monitor; gradually increase |
| **Rollback** | `kubectl rollout undo deployment/payment-service` |
| **Image tagging** | Use git SHA, not `latest` — immutable, traceable |

**What to say in the interview (4-beat answer)**
1. **Definition:** CI builds and tests on every commit; CD deploys to staging automatically and to production with approval. Tools: Jenkins (Groovy pipeline), GitHub Actions (YAML workflows), Docker for packaging, Kubernetes/OpenShift for deployment.
2. **Why/when:** CI catches bugs early (every commit triggers build + tests). CD ensures what's tested is what's deployed — same Docker image from build to prod. No "it works on my machine."
3. **Example:** Push to `main` → Jenkins builds → runs tests + SonarQube → builds Docker image tagged with git SHA → pushes to registry → deploys to staging → manual approval → deploys to production with rolling update.
4. **Gotcha/tradeoff:** Using `latest` tag for Docker images is non-reproducible — use git SHA or semver. Manual approval gates are necessary for production in banking (audit trail). Canary deployments reduce blast radius of bad releases.

**Common pitfalls**
- `docker:latest` tag — not immutable; can't reproduce or rollback.
- No test stage in CI — deploying untested code.
- No manual approval for production — compliance and audit failure in banking.
- Build on developer's machine → deploy — inconsistent environments; always build in CI.
- No rollback strategy — "fix forward" isn't always possible; `kubectl rollout undo` should be practiced.

**Self-check question**
Your CI pipeline takes 20 minutes. Developers push 30 times per day. How do you speed this up without skipping tests? Name three strategies.
