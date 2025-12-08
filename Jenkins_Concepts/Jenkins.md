# Jenkins and CI/CD Interview Questions and Answers

## Table of Contents

1. [CI/CD Fundamentals](#cicd-fundamentals)
2. [Jenkins Architecture](#jenkins-architecture)
3. [Jenkins Pipelines](#jenkins-pipelines)
4. [Jenkins Configuration](#jenkins-configuration)
5. [Jenkins Security](#jenkins-security)
6. [GitHub Actions](#github-actions)
7. [Deployment Strategies](#deployment-strategies)

---

## CI/CD Fundamentals

### Q1: Explain CI vs CD vs CD.

**Answer:**

| Term | Full Name | Description |
|------|-----------|-------------|
| CI | Continuous Integration | Merge code frequently, automated build/test |
| CD | Continuous Delivery | Auto-deploy to staging, manual prod approval |
| CD | Continuous Deployment | Auto-deploy to production (no manual gate) |

```
Code --> Build --> Test --> [Staging] --> [Approval] --> Production
        |--- CI ---|       |---- Continuous Delivery ----|
                           |---- Continuous Deployment -------|
```

---

### Q2: What is a typical CI/CD pipeline structure?

**Answer:**

| Stage | Purpose | Tools |
|-------|---------|-------|
| Source | Code checkout | Git, GitHub, GitLab |
| Build | Compile, package | Maven, Gradle, npm |
| Test | Unit, integration tests | JUnit, pytest, Jest |
| Security | SAST, dependency scan | SonarQube, Snyk, Trivy |
| Publish | Push artifacts/images | Docker Registry, Nexus |
| Deploy Dev | Automatic deployment | Kubernetes, Ansible |
| Deploy Staging | Environment testing | Kubernetes, Terraform |
| Deploy Prod | Production release | Kubernetes, ArgoCD |

---

## Jenkins Architecture

### Q3: Explain Jenkins architecture components.

**Answer:**

| Component | Description |
|-----------|-------------|
| Master | Orchestrates builds, serves UI, manages plugins |
| Agent (Node) | Executes builds, can be permanent or dynamic |
| Executor | Build slot on an agent (parallel capacity) |
| Job | Build configuration definition |
| Build | Single execution of a job |
| Workspace | Working directory for builds |
| Plugin | Extension adding functionality |

---

### Q4: What are Jenkins build agents and executors?

**Answer:**

**Agent types:**

| Type | Description | Use Case |
|------|-------------|----------|
| Permanent | Always connected | Dedicated build servers |
| Cloud/Dynamic | Created on-demand | Kubernetes, Docker, AWS |
| SSH | Connected via SSH | Linux agents |
| JNLP | Agent initiates connection | Behind firewall |

**Kubernetes dynamic agent example:**

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
                    image: maven:3.8-openjdk-17
                    command: ["sleep"]
                    args: ["infinity"]
                  - name: docker
                    image: docker:dind
                    securityContext:
                      privileged: true
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

---

## Jenkins Pipelines

### Q5: Explain Declarative vs Scripted pipelines.

**Answer:**

| Declarative | Scripted |
|-------------|----------|
| Structured with `pipeline {}` | Full Groovy with `node {}` |
| Easier to read | More flexible |
| Enforces structure | Any Groovy code |
| Better for most use cases | Complex conditional logic |

**Declarative Pipeline:**

```groovy
pipeline {
    agent any
    
    environment {
        REGISTRY = 'myregistry.io'
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }
    
    stages {
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
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh 'kubectl apply -f k8s/'
            }
        }
    }
    
    post {
        success {
            slackSend message: "Build succeeded"
        }
        failure {
            slackSend message: "Build failed"
        }
    }
}
```

**Scripted Pipeline:**

```groovy
node('linux') {
    try {
        stage('Build') {
            checkout scm
            sh 'mvn clean package'
        }
        
        stage('Test') {
            sh 'mvn test'
        }
        
        if (env.BRANCH_NAME == 'main') {
            stage('Deploy') {
                sh 'kubectl apply -f k8s/'
            }
        }
    } catch (Exception e) {
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        cleanWs()
    }
}
```

---

### Q6: How do you implement parallel stages?

**Answer:**

```groovy
pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }
        
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'mvn test -Dtest=*UnitTest'
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'mvn test -Dtest=*IntegrationTest'
                    }
                }
                stage('Lint') {
                    steps {
                        sh 'mvn checkstyle:check'
                    }
                }
            }
        }
    }
}
```

---

### Q7: What are Jenkins shared libraries?

**Answer:**

Shared libraries provide reusable code across pipelines.

**Directory structure:**

```
shared-library/
+-- vars/                    # Global variables/functions
|   +-- buildApp.groovy
|   +-- deployK8s.groovy
+-- src/                     # Groovy classes
|   +-- org/company/
+-- resources/               # Non-Groovy files
```

**vars/buildApp.groovy:**

```groovy
def call(Map config = [:]) {
    def language = config.language ?: 'java'
    
    pipeline {
        agent any
        stages {
            stage('Build') {
                steps {
                    script {
                        if (language == 'java') {
                            sh 'mvn clean package'
                        } else if (language == 'node') {
                            sh 'npm ci && npm run build'
                        }
                    }
                }
            }
        }
    }
}
```

**Using in Jenkinsfile:**

```groovy
@Library('shared-library@main') _

buildApp(language: 'java')
```

---

### Q8: How do you handle credentials in Jenkins?

**Answer:**

| Type | Use Case |
|------|----------|
| Username/Password | Basic auth, registry login |
| Secret text | API keys, tokens |
| Secret file | Kubeconfig, certificates |
| SSH private key | Git, server access |

```groovy
pipeline {
    agent any
    
    environment {
        DOCKER_CREDS = credentials('docker-hub')
    }
    
    stages {
        stage('Docker Login') {
            steps {
                sh '''
                    echo $DOCKER_CREDS_PSW | docker login \
                        -u $DOCKER_CREDS_USR --password-stdin
                '''
            }
        }
        
        stage('Use Secret File') {
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')
                ]) {
                    sh 'kubectl get pods'
                }
            }
        }
    }
}
```

---

### Q9: What is Jenkins Multibranch Pipeline?

**Answer:**

Automatically creates pipelines for each branch in a repository.

**Features:**
- Automatic branch discovery
- PR/MR builds
- Isolated builds per branch
- Automatic cleanup of deleted branches

```groovy
pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('Deploy to Dev') {
            when {
                branch 'develop'
            }
            steps {
                sh 'kubectl apply -f k8s/dev/'
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to production?"
            }
            steps {
                sh 'kubectl apply -f k8s/prod/'
            }
        }
        
        stage('PR Build') {
            when {
                changeRequest()
            }
            steps {
                echo "Building PR ${env.CHANGE_ID}"
            }
        }
    }
}
```

---

## Jenkins Configuration

### Q10: How do you optimize Jenkins pipeline performance?

**Answer:**

1. **Parallel execution**
2. **Dependency caching with persistent volumes**
3. **Docker layer caching with BuildKit**
4. **Skip stages with `when` conditions**
5. **Shallow clone for large repositories**

```groovy
options {
    skipDefaultCheckout()
}

stages {
    stage('Checkout') {
        steps {
            checkout([
                $class: 'GitSCM',
                branches: [[name: '*/main']],
                extensions: [[$class: 'CloneOption', depth: 1, shallow: true]],
                userRemoteConfigs: [[url: 'https://github.com/org/repo.git']]
            ])
        }
    }
}
```

---

### Q11: Explain Jenkins pipeline options and triggers.

**Answer:**

```groovy
pipeline {
    agent any
    
    options {
        timeout(time: 1, unit: 'HOURS')
        retry(3)
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }
    
    triggers {
        pollSCM('H/5 * * * *')
        cron('H 2 * * *')
        githubPush()
    }
    
    stages {
        // ...
    }
}
```

---

## Jenkins Security

### Q12: How do you secure Jenkins?

**Answer:**

1. **Authentication:** LDAP/AD, SSO (SAML, OAuth)
2. **Authorization:** Role-Based Access Control (RBAC)
3. **Credentials:** Use Jenkins Credentials Store, never hardcode
4. **Network:** Run behind reverse proxy with HTTPS
5. **Plugins:** Keep updated, remove unused
6. **Script security:** Enable sandbox, review approvals
7. **Audit logging:** Enable audit trail plugin

---

## GitHub Actions

### Q13: Compare Jenkins vs GitHub Actions.

**Answer:**

| Feature | Jenkins | GitHub Actions |
|---------|---------|----------------|
| Hosting | Self-hosted / Cloud | Cloud-native |
| Configuration | Groovy | YAML |
| Plugins | Large ecosystem | Marketplace Actions |
| Learning curve | Steeper | Easier |
| Git provider | Any | GitHub only |
| Cost | Free (self-hosted) | Free tier + paid |

---

### Q14: Explain GitHub Actions workflow syntax.

**Answer:**

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        default: 'staging'

env:
  REGISTRY: ghcr.io

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Build
        run: mvn clean package
      
      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar

  deploy:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
      - name: Deploy
        run: echo "Deploying..."
```

---

### Q15: How do you handle secrets in GitHub Actions?

**Answer:**

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Login to registry
        run: |
          echo ${{ secrets.DOCKER_PASSWORD }} | \
          docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
      
      - name: Configure AWS (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions
          aws-region: eu-west-1
```

---

## Deployment Strategies

### Q16: Explain different deployment strategies.

**Answer:**

| Strategy | Description | Risk | Rollback |
|----------|-------------|------|----------|
| Recreate | Stop old, start new | High | Redeploy |
| Rolling | Gradual replacement | Medium | Redeploy |
| Blue-Green | Two environments | Low | Switch traffic |
| Canary | Gradual traffic shift | Low | Route to old |

**Rolling Update (Kubernetes):**

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```

**Canary (Argo Rollouts):**

```yaml
spec:
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: {duration: 5m}
        - setWeight: 50
        - pause: {duration: 5m}
        - setWeight: 100
```

---

### Q17: What is GitOps?

**Answer:**

GitOps uses Git as the single source of truth for infrastructure.

**Principles:**
1. Declarative - System described declaratively
2. Versioned - Desired state in Git
3. Automated - Changes auto-applied
4. Reconciliation - Continuous sync

**Argo CD Application:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  source:
    repoURL: https://github.com/org/gitops-repo
    path: apps/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## Summary

| Topic | Key Concepts |
|-------|--------------|
| CI/CD | Integration, delivery, deployment |
| Jenkins | Master, agents, pipelines, plugins |
| Pipelines | Declarative, scripted, shared libraries |
| Security | Credentials, RBAC, audit |
| GitHub Actions | Workflows, actions, secrets |
| Deployment | Rolling, blue-green, canary |
| GitOps | Git as source of truth |

**Essential Jenkins pipeline structure:**

```groovy
pipeline {
    agent { }           // Where to run
    environment { }     // Variables
    options { }         // Pipeline options
    triggers { }        // Build triggers
    stages { }          // Build stages
    post { }            // Post-build actions
}
```