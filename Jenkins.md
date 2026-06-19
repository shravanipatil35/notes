# Jenkins — Complete Notes (Basics → Advanced + Interview Prep)

---

## Table of Contents
1. [Introduction](#1-introduction)
2. [Jenkins vs Other CI/CD Tools](#2-jenkins-vs-other-cicd-tools)
3. [Jenkins Architecture](#3-jenkins-architecture)
4. [Installation & Setup](#4-installation--setup)
5. [Core Concepts](#5-core-concepts)
6. [Jobs / Build Types](#6-jobs--build-types)
7. [Pipelines — Declarative Syntax](#7-pipelines--declarative-syntax)
8. [Pipelines — Scripted Syntax](#8-pipelines--scripted-syntax)
9. [Pipeline Stages, Steps & Directives](#9-pipeline-stages-steps--directives)
10. [Agents & Distributed Builds](#10-agents--distributed-builds)
11. [Triggers](#11-triggers)
12. [Parameters](#12-parameters)
13. [Environment Variables & Credentials](#13-environment-variables--credentials)
14. [Plugins](#14-plugins)
15. [Shared Libraries](#15-shared-libraries)
16. [Multibranch Pipelines](#16-multibranch-pipelines)
17. [Post Actions & Notifications](#17-post-actions--notifications)
18. [Artifacts & Test Reports](#18-artifacts--test-reports)
19. [Jenkinsfile Best Practices](#19-jenkinsfile-best-practices)
20. [Jenkins with Docker & Kubernetes](#20-jenkins-with-docker--kubernetes)
21. [Security](#21-security)
22. [Backup, Scaling & High Availability](#22-backup-scaling--high-availability)
23. [Blue Ocean & Jenkins X](#23-blue-ocean--jenkins-x)
24. [Troubleshooting](#24-troubleshooting)
25. [Best Practices Summary](#25-best-practices-summary)
26. [Cheat Sheet](#26-cheat-sheet)
27. [Interview Questions & Answers](#27-interview-questions--answers)

---

## 1. Introduction

**Jenkins** is an open-source **automation server** used primarily for building **CI/CD (Continuous Integration / Continuous Delivery/Deployment)** pipelines. Originally forked from Hudson in 2011, it's written in Java and is one of the most widely adopted automation tools in the industry due to its huge plugin ecosystem (1800+ plugins) and flexibility.

**What Jenkins automates:**
- Pulling code from version control (Git, SVN, etc.)
- Building/compiling code
- Running automated tests
- Static analysis / code quality checks
- Packaging artifacts (jars, Docker images, etc.)
- Deploying to staging/production environments
- Sending notifications (Slack, email) on build status

**Why Jenkins?**
- Free, open-source, highly extensible via plugins
- Supports virtually any language/build tool/platform
- Pipeline-as-code (Jenkinsfile) — version-controlled, reviewable CI/CD definitions
- Master-agent architecture scales builds across many machines
- Large community and integration ecosystem

---

## 2. Jenkins vs Other CI/CD Tools

| Aspect | Jenkins | GitHub Actions | GitLab CI | CircleCI |
|---|---|---|---|---|
| Hosting | Self-hosted (or cloud) | Cloud-native (GitHub) | Cloud or self-hosted | Cloud-native |
| Config | Jenkinsfile (Groovy) | YAML workflow files | `.gitlab-ci.yml` | `.circleci/config.yml` |
| Setup overhead | High (you manage the server) | Low (built into GitHub) | Low-Medium | Low |
| Flexibility | Extremely high (plugins) | Moderate (marketplace actions) | High | Moderate |
| Scaling | Manual (agents/cloud plugins) | Managed by GitHub | Managed or self-hosted runners | Managed |
| Best for | Complex, custom, legacy/enterprise pipelines | Repos already on GitHub | Repos already on GitLab | Simplicity-focused teams |

**Key interview point:** Jenkins' main strength is flexibility and control (self-hosted, massive plugin ecosystem, works with any SCM/platform) at the cost of operational overhead (you maintain the server, agents, plugins, upgrades). Cloud-native tools (GitHub Actions, GitLab CI) trade some flexibility for zero infrastructure management.

---

## 3. Jenkins Architecture

```
                    ┌─────────────────────────────┐
                    │      Jenkins Controller        │
                    │      (Master)                   │
                    │  - Web UI / REST API             │
                    │  - Schedules builds               │
                    │  - Stores job configs              │
                    │  - Dispatches work to agents         │
                    └──────────────┬───────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
      ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
      │   Agent 1        │   │   Agent 2        │   │   Agent 3        │
      │ (Linux build      │   │ (Windows build    │   │ (Docker/K8s        │
      │  executor)          │   │  executor)          │   │  ephemeral pod)     │
      └───────────────┘   └───────────────┘   └───────────────┘
```

**Components:**
- **Controller (Master)** — the central Jenkins server: hosts the web UI, schedules jobs, stores configuration, dispatches build tasks to agents. (Older docs call this the "master.")
- **Agent (Node/Slave)** — a machine (physical, VM, container, cloud instance) that actually executes build steps. Agents connect to the controller via JNLP, SSH, or other protocols.
- **Executor** — a slot on an agent that can run one build at a time; an agent can have multiple executors for parallel builds.
- **Workspace** — a directory on the agent where the build's files (checked-out source, build outputs) live during execution.

**Why distribute builds to agents?** Running everything on the controller doesn't scale and is a security risk (build code/plugins running with controller-level access). Distributing to agents enables parallelism, isolates untrusted build code, and lets you match builds to specific environments (e.g., a Windows agent for .NET builds, a GPU agent for ML jobs).

---

## 4. Installation & Setup

```bash
# Docker (quickest way to try Jenkins)
docker run -d -p 8080:8080 -p 50000:50000 --name jenkins \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts

# Linux (Debian/Ubuntu) native install
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
sudo systemctl start jenkins
```

**Initial setup:**
```bash
# Get the initial admin password
cat /var/lib/jenkins/secrets/initialAdminPassword
```
Then browse to `http://localhost:8080`, unlock with that password, install suggested plugins, create an admin user.

**Key directories:**
- `JENKINS_HOME` (e.g., `/var/lib/jenkins`) — all configuration, job definitions, plugins, build history, secrets
- `JENKINS_HOME/jobs/<job-name>` — per-job config and build history

---

## 5. Core Concepts

| Term | Definition |
|---|---|
| **Job/Project** | A configured task Jenkins can run (e.g., build, test) |
| **Build** | A single execution/run of a job |
| **Pipeline** | A suite of steps (often defined in a Jenkinsfile) describing the full CI/CD process as code |
| **Jenkinsfile** | A text file (Groovy-based DSL) defining a Pipeline, usually checked into source control |
| **Stage** | A logical, distinct phase of a pipeline (e.g., Build, Test, Deploy) shown in the UI |
| **Step** | A single task within a stage (e.g., run a shell command) |
| **Agent/Node** | A machine that executes builds |
| **Executor** | A build slot on an agent |
| **Plugin** | Extends Jenkins functionality (SCM integrations, notifications, build tools, etc.) |
| **Trigger** | What causes a build to start (SCM webhook, schedule, manual, upstream job) |
| **Artifact** | A file produced by a build, archived for later use/download |

---

## 6. Jobs / Build Types

| Job Type | Description |
|---|---|
| **Freestyle project** | Configured entirely via the web UI (no code) — simple but not version-controlled, harder to scale/maintain |
| **Pipeline** | Defined via a Jenkinsfile (Groovy DSL) — code-based, version-controlled, supports complex logic |
| **Multibranch Pipeline** | Automatically creates a sub-job for each branch (and PR) in a repo that has a Jenkinsfile |
| **Folder** | Organizational grouping for related jobs |
| **Multi-configuration (Matrix) project** | Runs the same job across a matrix of configurations (e.g., multiple OS/JDK combos) |

**Freestyle vs Pipeline (common interview question):** Freestyle jobs are configured via UI clicks, are not version-controlled by default, and don't handle complex logic well. Pipeline jobs are defined as code (Jenkinsfile) checked into the repo alongside the application — giving you code review, versioning, reusability, and far more powerful control flow (stages, parallelism, error handling). Pipeline is the modern, recommended approach.

---

## 7. Pipelines — Declarative Syntax

The modern, structured, **recommended** way to write a Jenkinsfile.

```groovy
pipeline {
    agent any

    environment {
        APP_ENV = 'production'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }

    parameters {
        string(name: 'BRANCH', defaultValue: 'main', description: 'Branch to build')
    }

    triggers {
        pollSCM('H/5 * * * *')
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${params.BRANCH}", url: 'https://github.com/example/app.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh './deploy.sh'
            }
        }
    }

    post {
        success {
            echo 'Build succeeded!'
        }
        failure {
            mail to: 'team@example.com', subject: 'Build Failed', body: 'Check Jenkins.'
        }
        always {
            cleanWs()
        }
    }
}
```

---

## 8. Pipelines — Scripted Syntax

Older, more flexible (full Groovy programming power), but less structured and harder to read/maintain. Generally used only when Declarative's structure is too limiting.

```groovy
node {
    stage('Checkout') {
        git url: 'https://github.com/example/app.git'
    }
    stage('Build') {
        sh 'mvn clean package'
    }
    stage('Test') {
        try {
            sh 'mvn test'
        } catch (Exception e) {
            currentBuild.result = 'UNSTABLE'
        }
    }
    stage('Deploy') {
        if (env.BRANCH_NAME == 'main') {
            sh './deploy.sh'
        }
    }
}
```

**Declarative vs Scripted (common interview question):** Declarative has a fixed, structured syntax (`pipeline { agent {} stages {} }`) that's easier to read, validate, and lint, and is the recommended default. Scripted is essentially raw Groovy with Jenkins-specific steps — more flexible/powerful for complex conditional logic but more error-prone and harder for teams to standardize on. Declarative pipelines can drop into Scripted blocks using the `script {}` step when needed.

---

## 9. Pipeline Stages, Steps & Directives

**Key Declarative directives:**
| Directive | Purpose |
|---|---|
| `agent` | Where the pipeline/stage runs (`any`, `none`, `label`, `docker`, `kubernetes`) |
| `stages` | Container for all `stage` blocks |
| `stage` | A logical phase of the pipeline |
| `steps` | Actual commands/actions within a stage |
| `environment` | Define env vars (pipeline- or stage-scoped) |
| `parameters` | Define build input parameters |
| `triggers` | Define what starts a build automatically |
| `options` | Pipeline-level options (timeout, retry, etc.) |
| `post` | Actions to run after stage/pipeline completion based on status |
| `when` | Conditional execution of a stage |
| `parallel` | Run stages/steps concurrently |
| `input` | Pause and wait for manual approval |

**Parallel execution:**
```groovy
stage('Tests') {
    parallel {
        stage('Unit Tests') {
            steps { sh 'mvn test' }
        }
        stage('Integration Tests') {
            steps { sh 'mvn verify' }
        }
    }
}
```

**Manual approval gate:**
```groovy
stage('Deploy to Prod') {
    steps {
        input message: 'Deploy to production?', ok: 'Deploy'
        sh './deploy-prod.sh'
    }
}
```

**Conditional stage (`when`):**
```groovy
stage('Deploy') {
    when {
        allOf {
            branch 'main'
            expression { return params.DEPLOY == true }
        }
    }
    steps {
        sh './deploy.sh'
    }
}
```

---

## 10. Agents & Distributed Builds

```groovy
// Run on any available agent
pipeline {
    agent any
    ...
}

// Run on an agent with a specific label
pipeline {
    agent { label 'linux && docker' }
    ...
}

// Run inside a Docker container
pipeline {
    agent {
        docker { image 'maven:3.9-eclipse-temurin-17' }
    }
    ...
}

// Different agent per stage
pipeline {
    agent none
    stages {
        stage('Build') {
            agent { label 'linux' }
            steps { sh 'make build' }
        }
        stage('Test on Windows') {
            agent { label 'windows' }
            steps { bat 'run-tests.bat' }
        }
    }
}
```

**Connecting agents:** via SSH (Jenkins controller connects out to the agent), JNLP/Inbound (agent connects in to the controller — useful behind firewalls/NAT), or dynamically via cloud plugins (Kubernetes plugin, EC2 plugin, Docker plugin) that spin up ephemeral agents per build and tear them down after.

---

## 11. Triggers

| Trigger | Description |
|---|---|
| **SCM Webhook** | Repo (GitHub/GitLab/Bitbucket) pushes a webhook to Jenkins on code change — fastest, push-based |
| **Poll SCM** (`pollSCM`) | Jenkins periodically checks the repo for changes (cron syntax) — fallback when webhooks aren't possible |
| **Cron/Build periodically** (`cron`) | Runs on a fixed schedule regardless of changes |
| **Upstream/downstream** | Triggered after another job completes |
| **Manual** | Triggered by a person clicking "Build Now" |
| **Remote API trigger** | Triggered via HTTP POST to Jenkins' REST API (e.g., from another system) |

```groovy
triggers {
    githubPush()             // webhook-based
    pollSCM('H/5 * * * *')   // poll every ~5 minutes
    cron('H 2 * * *')        // nightly build ~2 AM
}
```

**Webhook vs Poll SCM (interview point):** Webhooks are push-based — the SCM notifies Jenkins instantly on a change, faster and more efficient. Polling is pull-based — Jenkins repeatedly checks for changes on an interval, wasting resources and introducing delay, but useful when Jenkins isn't reachable from the internet/SCM (no inbound webhook possible).

---

## 12. Parameters

```groovy
parameters {
    string(name: 'VERSION', defaultValue: '1.0.0', description: 'Version to build')
    booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Run test suite')
    choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Target environment')
    password(name: 'API_KEY', defaultValue: '', description: 'API key')
}

stages {
    stage('Deploy') {
        steps {
            echo "Deploying version ${params.VERSION} to ${params.ENVIRONMENT}"
        }
    }
}
```
Parameters appear as a form when triggering "Build with Parameters" in the UI, and can also be passed via the REST API/CLI.

---

## 13. Environment Variables & Credentials

```groovy
environment {
    APP_ENV = 'production'
    DB_URL  = "jdbc:mysql://${params.DB_HOST}/mydb"
}
```

**Built-in env vars:** `BUILD_NUMBER`, `BUILD_ID`, `JOB_NAME`, `WORKSPACE`, `BRANCH_NAME` (multibranch), `GIT_COMMIT`, `JENKINS_URL`.

**Credentials (never hardcode secrets in a Jenkinsfile):**
```groovy
environment {
    DOCKER_CREDS = credentials('dockerhub-creds')   // injects DOCKER_CREDS_USR / DOCKER_CREDS_PSW
}

stages {
    stage('Login') {
        steps {
            sh 'echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin'
        }
    }
}
```
```groovy
// withCredentials block — scoped injection
withCredentials([usernamePassword(credentialsId: 'db-creds', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS')]) {
    sh './migrate.sh'
}
```
Credentials are stored in Jenkins' encrypted **Credentials Store** (Manage Jenkins → Credentials), referenced by ID — never stored in plaintext in the Jenkinsfile itself. Supports username/password, SSH keys, secret text, certificates.

---

## 14. Plugins

Jenkins' core is minimal; almost all functionality (Git integration, Docker support, Slack notifications, Kubernetes agents, code coverage reports) comes from **plugins** (1800+ available).

**Commonly used plugins:**
- **Git / GitHub / GitLab** — SCM integration
- **Pipeline** — core Pipeline-as-code support (usually pre-installed)
- **Docker Pipeline** — build/run Docker containers from pipelines
- **Kubernetes** — dynamically provision build agents as K8s pods
- **Credentials Binding** — secure secret injection
- **Blue Ocean** — modern visual pipeline UI
- **Slack Notification** — send build status to Slack
- **JUnit** — parse/display test results
- **SonarQube Scanner** — code quality integration
- **Artifactory/Nexus** — artifact repository integration

```bash
# Manage Jenkins → Plugins, or via CLI:
jenkins-plugin-cli --plugins git docker-workflow kubernetes
```

---

## 15. Shared Libraries

Reusable Groovy code shared across multiple Jenkinsfiles/pipelines — avoids duplicating common logic (e.g., a standard "build and push Docker image" routine) across dozens of repos.

```
(shared-library-repo)/
  vars/
    buildAndPush.groovy
  src/
    org/example/Utils.groovy
```

```groovy
// vars/buildAndPush.groovy
def call(String imageName) {
    sh "docker build -t ${imageName} ."
    sh "docker push ${imageName}"
}
```

```groovy
// Using it in a Jenkinsfile
@Library('my-shared-library') _

pipeline {
    agent any
    stages {
        stage('Build & Push') {
            steps {
                buildAndPush('myrepo/myapp:1.0')
            }
        }
    }
}
```
Configured globally via Manage Jenkins → System → Global Pipeline Libraries, pointing to a Git repo.

---

## 16. Multibranch Pipelines

Automatically discovers branches (and pull requests) in a repository and creates/manages a sub-pipeline for each one that contains a Jenkinsfile — deleting the corresponding job automatically when a branch is deleted.

**Benefits:**
- No manual job creation per branch/feature branch
- Each branch can have its own Jenkinsfile with different logic if needed
- Naturally supports PR validation builds (build + test on every PR before merge)

```groovy
// Branch-specific logic inside one shared Jenkinsfile
when {
    branch 'main'
}
```

Configured by pointing a "Multibranch Pipeline" job at a repo — Jenkins scans it periodically (or via webhook) for branches containing a `Jenkinsfile`.

---

## 17. Post Actions & Notifications

```groovy
post {
    always {
        echo 'This always runs'
        cleanWs()
    }
    success {
        slackSend channel: '#builds', message: "✅ Build #${BUILD_NUMBER} succeeded"
    }
    failure {
        slackSend channel: '#builds', message: "❌ Build #${BUILD_NUMBER} failed"
        mail to: 'team@example.com', subject: "Build Failed: ${JOB_NAME}", body: "${BUILD_URL}"
    }
    unstable {
        echo 'Build is unstable (e.g., test failures but not a hard failure)'
    }
    changed {
        echo 'Status changed from the previous build (e.g., fixed or newly broken)'
    }
}
```

---

## 18. Artifacts & Test Reports

```groovy
stage('Archive') {
    steps {
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
    }
}

stage('Test') {
    steps {
        sh 'mvn test'
    }
    post {
        always {
            junit 'target/surefire-reports/*.xml'
            publishHTML(target: [
                reportDir: 'coverage',
                reportFiles: 'index.html',
                reportName: 'Coverage Report'
            ])
        }
    }
}
```
- `archiveArtifacts` — saves build output files, downloadable from the build page, with optional fingerprinting (tracks artifact usage across jobs)
- `junit` — parses XML test reports, shows pass/fail trends and graphs in the UI
- Integrations with Nexus/Artifactory/S3 for longer-term artifact storage beyond Jenkins' own retention

---

## 19. Jenkinsfile Best Practices

- Store the **Jenkinsfile in source control**, alongside the application code
- Prefer **Declarative** syntax; drop into `script {}` only for logic Declarative can't express
- Keep stages **focused and named clearly** (Checkout, Build, Test, Scan, Deploy)
- Use **parallel** stages for independent, time-consuming steps (e.g., unit tests + lint + security scan)
- Externalize repeated logic into **Shared Libraries**
- Use **`when`** conditions instead of branching logic scattered with `if` statements
- Set a pipeline-level **`timeout`** to avoid stuck builds consuming executors indefinitely
- Use **`input`** steps for manual approval gates before production deploys
- Clean workspaces (`cleanWs()`) in `post { always {} }` to avoid disk bloat
- Never hardcode secrets — always use the Credentials store

---

## 20. Jenkins with Docker & Kubernetes

**Building Docker images in a pipeline:**
```groovy
pipeline {
    agent any
    stages {
        stage('Build Image') {
            steps {
                script {
                    docker.build("myapp:${env.BUILD_NUMBER}")
                }
            }
        }
        stage('Push Image') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-creds') {
                        docker.image("myapp:${env.BUILD_NUMBER}").push()
                    }
                }
            }
        }
    }
}
```

**Dynamic Kubernetes agents** (Kubernetes plugin) — spins up a fresh pod per build, perfect isolation, auto-scales agents to zero when idle:
```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
              apiVersion: v1
              kind: Pod
              spec:
                containers:
                - name: maven
                  image: maven:3.9-eclipse-temurin-17
                  command: ["cat"]
                  tty: true
            '''
        }
    }
    stages {
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package'
                }
            }
        }
    }
}
```
This pattern (Jenkins controller running in K8s, dynamically scheduling per-build agent pods) is very common in modern cloud-native setups — it avoids maintaining a static fleet of agent VMs and isolates every build in a fresh, ephemeral environment.

---

## 21. Security

1. **Enable security realm & authorization** (Matrix-based or Role-based Authorization Strategy) — never leave Jenkins open/anonymous in production.
2. **Use the Credentials store** for all secrets — never hardcode in Jenkinsfiles or job configs.
3. **Restrict script approval** — Groovy sandbox blocks dangerous operations in untrusted Pipeline scripts by default; approve only what's necessary via Manage Jenkins → In-process Script Approval.
4. **Run agents with least privilege**, isolate untrusted code execution to agents (not the controller).
5. **Keep Jenkins core and plugins updated** — many historical CVEs target outdated plugins.
6. **Use HTTPS** for the Jenkins web UI.
7. **Audit logging** — track who changed configs/triggered builds (Audit Trail plugin).
8. **Limit "Build Now" / webhook trigger access** to authorized branches/users to prevent pipeline abuse.
9. **Use folder-level permissions** to scope access for multi-team Jenkins instances.
10. **Avoid running pipeline steps directly on the controller** when they involve untrusted/build-specific code — use agents.

---

## 22. Backup, Scaling & High Availability

- **Backup `JENKINS_HOME`** regularly (job configs, plugins, credentials, build history) — e.g., via the ThinBackup plugin or simple filesystem/volume snapshots.
- **Configuration as Code (JCasC) plugin** — define Jenkins' own system configuration in YAML, version-controlled, enabling reproducible Jenkins instances.
- **Horizontal scaling** — add more agents (static or dynamic via cloud/K8s plugins) rather than scaling the controller; the controller itself is traditionally a scaling bottleneck/single point of failure.
- **High availability** — typically achieved via active/passive failover setups or by treating the controller as disposable (JCasC + backed-up `JENKINS_HOME` + automated redeploy) rather than true active-active clustering, which Jenkins doesn't natively support well.

---

## 23. Blue Ocean & Jenkins X

- **Blue Ocean** — a modern, visual UI plugin for Jenkins Pipelines, showing pipeline stages/steps as a clear visual flow graph instead of the classic console-log-heavy UI. (Note: Blue Ocean has been deprecated/archived as of recent Jenkins releases — worth a quick check of current status if relevant to your environment.)
- **Jenkins X** — a separate, Kubernetes-native CI/CD project built around Jenkins concepts but designed specifically for cloud-native, GitOps-style workflows on Kubernetes (different from classic Jenkins; uses Tekton pipelines under the hood in newer versions).

---

## 24. Troubleshooting

```bash
# Check Jenkins logs
sudo journalctl -u jenkins -f          # systemd-managed installs
docker logs -f jenkins                  # Docker-based installs

# Inside Jenkins
Manage Jenkins → System Log              # web UI log viewer
Manage Jenkins → Script Console          # run Groovy directly against the running instance for debugging
```

**Common issues:**
| Issue | Cause / Fix |
|---|---|
| Build stuck "pending" forever | No available executor matching the required agent label — check agent connectivity/labels |
| `Permission denied` on agent | Credentials misconfigured, or agent user lacks file/docker permissions |
| Pipeline syntax error | Use "Replay" or the Pipeline Syntax generator in the UI; validate with `Jenkinsfile Linter` (`curl` to `/pipeline-model-converter/validate`) |
| Webhook not triggering build | Check repo webhook delivery logs, Jenkins URL reachability, and that the Multibranch/job is configured to scan on push |
| Out of disk space on controller | Old builds/artifacts not being discarded — configure build/artifact retention (`buildDiscarder`) |
| Groovy `RejectedAccessException` | Script Security sandbox blocking an operation — approve via Script Approval (carefully, only for trusted code) |

```groovy
options {
    buildDiscarder(logRotator(numToKeepStr: '30'))   // auto-delete old builds
}
```

---

## 25. Best Practices Summary

- Use Pipeline (Jenkinsfile) over Freestyle jobs for anything beyond trivial tasks
- Keep Jenkinsfiles in source control, next to the code they build
- Prefer Declarative syntax; minimize raw `script{}` blocks
- Use dynamic agents (Docker/Kubernetes plugin) over static long-lived agents where possible
- Store all secrets in the Credentials store, never inline
- Use Shared Libraries to DRY up common pipeline logic across teams/repos
- Set timeouts and build/artifact retention policies
- Use Multibranch Pipelines for repos with active branching/PR workflows
- Gate production deploys with manual `input` approval steps
- Keep the controller free of untrusted build execution; isolate to agents
- Regularly back up `JENKINS_HOME` and manage Jenkins config via JCasC where possible
- Keep core + plugins patched and up to date

---

## 26. Cheat Sheet

```groovy
pipeline {
    agent any
    options { timeout(time: 30, unit: 'MINUTES') }
    parameters { string(name: 'X', defaultValue: 'y', description: '') }
    triggers { pollSCM('H/5 * * * *') }
    environment { KEY = 'value' }
    stages {
        stage('Name') {
            when { branch 'main' }
            steps {
                sh 'command'
            }
            post { always { echo 'done' } }
        }
    }
    post {
        success { echo 'ok' }
        failure { echo 'failed' }
        always { cleanWs() }
    }
}
```

```bash
# CLI / common ops
java -jar jenkins-cli.jar -s http://localhost:8080/ build <job-name>
curl -X POST JENKINS_URL/job/JOB_NAME/build --user user:apitoken
cat /var/lib/jenkins/secrets/initialAdminPassword
```

---

## 27. Interview Questions & Answers

**Q1: What is Jenkins and what is it used for?**
A: Jenkins is an open-source automation server primarily used to build CI/CD pipelines — automating the build, test, and deployment of software whenever code changes, using either UI-configured jobs or, more commonly today, code-defined Pipelines (Jenkinsfiles).

**Q2: Explain Jenkins' controller-agent architecture.**
A: The **controller** (master) hosts the web UI/API, schedules builds, and stores configuration, but doesn't necessarily run build work itself in production setups. **Agents** (nodes) are separate machines/containers that actually execute build steps, connecting to the controller via SSH/JNLP or being dynamically provisioned (e.g., via the Kubernetes plugin). This separates orchestration from execution, enabling parallelism, environment-specific builds, and isolation of untrusted build code from the controller.

**Q3: Difference between Declarative and Scripted Pipeline syntax?**
A: Declarative uses a fixed, structured format (`pipeline { agent {} stages {} }`) — easier to read, lint, and standardize across teams, and is the recommended default. Scripted is raw Groovy with Jenkins steps — more flexible for complex control flow but harder to maintain and validate. Declarative pipelines can embed a `script {}` block to drop into Scripted-style code when needed.

**Q4: What is a Jenkinsfile and why check it into source control?**
A: A Jenkinsfile is a text file (Groovy DSL) defining a Pipeline's stages and steps. Storing it in source control alongside the application code gives you version history, code review via pull requests, and ensures the CI/CD process evolves together with the codebase (and automatically per-branch in Multibranch setups).

**Q5: How do you handle secrets/credentials in a Jenkins pipeline?**
A: Store them in Jenkins' encrypted Credentials store (never hardcoded in the Jenkinsfile), then reference them by ID — either via the `environment { X = credentials('id') }` block or the `withCredentials([...]) {}` step, which injects them as scoped environment variables only for the duration needed.

**Q6: What is a Multibranch Pipeline?**
A: A job type that automatically scans a repository for branches/PRs containing a Jenkinsfile and creates/manages a corresponding sub-pipeline for each one — removing the need to manually create a Jenkins job per branch, and automatically cleaning up jobs when branches are deleted.

**Q7: What's the difference between `pollSCM` and a webhook trigger?**
A: A webhook is push-based — the SCM (e.g., GitHub) notifies Jenkins instantly when a change happens, which is fast and efficient. `pollSCM` is pull-based — Jenkins checks the repo for changes on a cron schedule, introducing delay and unnecessary load, but useful as a fallback when Jenkins can't receive inbound webhooks (e.g., behind a firewall).

**Q8: What are Shared Libraries and why use them?**
A: Reusable Groovy code (stored in a separate Git repo) that multiple Jenkinsfiles can import via `@Library`, used to avoid duplicating common pipeline logic (e.g., a standard build-and-push routine) across many projects — improving consistency and maintainability across an organization's pipelines.

**Q9: How would you run parallel stages in a pipeline, and why?**
A: Using the `parallel {}` block to run independent stages (e.g., unit tests, linting, security scanning) concurrently rather than sequentially — reducing total pipeline runtime when steps don't depend on each other's output.

**Q10: How do you implement a manual approval gate before deploying to production?**
A: Use the `input` step within a stage (e.g., `input message: 'Deploy to production?'`), which pauses pipeline execution and waits for a user with appropriate permissions to approve (or abort) before continuing — commonly placed right before a production deployment stage.

**Q11: What is the Jenkins Credentials store, and how does it differ from just using environment variables?**
A: The Credentials store is an encrypted, access-controlled location for secrets (passwords, SSH keys, tokens, certificates) managed by Jenkins, referenced by ID rather than embedded as plaintext. Plain environment variables set directly in a Jenkinsfile would be visible in the pipeline source/logs — the Credentials store keeps secret values out of source control and masks them in console output.

**Q12: How do you scale Jenkins to handle many concurrent builds?**
A: Add more build agents — either statically provisioned VMs/machines, or dynamically via cloud/Kubernetes plugins that spin up ephemeral agent pods per build and tear them down afterward, auto-scaling agent capacity with load. The controller itself should be kept lightweight (orchestration only), since it's traditionally harder to horizontally scale.

**Q13: What is the difference between a Freestyle job and a Pipeline job?**
A: A Freestyle job is configured entirely through the web UI — simple to set up for basic tasks but not version-controlled and limited in expressing complex logic. A Pipeline job is defined as code (Jenkinsfile), supports complex control flow (stages, parallelism, conditionals, error handling), and is version-controlled alongside the application — the modern recommended approach for anything beyond trivial automation.

**Q14: How does Jenkins integrate with Docker?**
A: Jenkins can build Docker images as a pipeline step (`docker.build(...)`), push them to a registry (`docker.withRegistry(...)`), and also use Docker as the **execution environment** for builds themselves — running each build inside a specified container image via `agent { docker { image '...' } }`, ensuring a clean, consistent, dependency-isolated build environment per pipeline run.

**Q15: What happens if a pipeline stage fails? How do you control failure behavior?**
A: By default, a failing step marks the build as `FAILURE` and (in Declarative) skips subsequent stages, though `post { always/failure/success }` blocks still run for cleanup/notification. You can control behavior with `catchError`, `try/catch` (in `script{}` blocks), the `unstable()` step (marks build unstable rather than fully failed, e.g., on test failures), and per-stage `post` blocks for stage-specific handling.

**Q16: What's the role of the Groovy Sandbox / Script Security plugin?**
A: It restricts what untrusted Pipeline scripts (e.g., from a Jenkinsfile in a regular, non-admin-controlled repo) are allowed to do — blocking potentially dangerous Groovy operations (like arbitrary system calls) unless explicitly approved by an administrator via Script Approval, protecting the controller from malicious or accidental script behavior.

**Q17: How do you avoid hardcoding environment-specific values across dev/staging/prod pipelines?**
A: Use pipeline `parameters` for user-driven choices, `environment` blocks combined with conditional logic (`when` / branch-based logic) for environment-specific values, externalize configuration into separate files/credentials per environment, and leverage Multibranch Pipelines or separate Jenkinsfiles per environment branch where logic genuinely diverges.

**Q18: What is Jenkins Configuration as Code (JCasC)?**
A: A plugin that lets you define Jenkins' own system-level configuration (security settings, plugin config, credentials references, agent definitions, etc.) in version-controlled YAML files instead of manual UI clicks — enabling reproducible, disaster-recoverable Jenkins instances that can be redeployed from code.

**Q19: How would you debug a pipeline that's stuck in a "pending" state?**
A: Check whether there's an available agent matching the pipeline's required `agent` label — a stuck/pending build commonly means no executor is currently free or no agent matches the label constraint; also check agent connectivity (offline agents), and review the build queue in the Jenkins UI for the specific reason it's waiting.

**Q20: What's the benefit of using ephemeral Kubernetes-based build agents over static agent VMs?**
A: Each build gets a completely fresh, isolated environment (no leftover state/dependency drift between builds), agent capacity auto-scales with demand (scaling to zero when idle, saving cost), and there's no ongoing OS/agent-software maintenance burden compared to a static fleet of long-lived VMs.

---

### Final interview tip
Be ready to write a basic **Declarative Jenkinsfile from scratch** (checkout → build → test → deploy with a `post` block), explain the **controller-agent model** and why builds shouldn't run on the controller, and clearly articulate **Declarative vs Scripted** and **webhook vs polling** trade-offs — these are asked constantly. Also expect a practical "how would you secure your Jenkins pipeline's secrets" question.
