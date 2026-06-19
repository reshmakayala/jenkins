# Jenkins Cheat Sheet — Part 2

## 6. Credentials Management

Proper credential management is critical for secure CI/CD pipelines.

---

### Supported Credential Types

| Type                          | Use Case                   |
| ----------------------------- | -------------------------- |
| Username with Password        | Git, Docker Registry, APIs |
| SSH Username with Private Key | Git SSH Authentication     |
| Secret Text                   | Tokens, API Keys           |
| Secret File                   | Certificates, Kubeconfig   |
| Certificate                   | TLS/SSL Authentication     |

---

### Creating Credentials

```text
Manage Jenkins
→ Credentials
→ System
→ Global Credentials
→ Add Credentials
```

---

### Username/Password Example

Credential ID:

```text
github-creds
```

Pipeline:

```groovy
withCredentials([
    usernamePassword(
        credentialsId: 'github-creds',
        usernameVariable: 'USER',
        passwordVariable: 'PASS'
    )
]) {

    sh '''
        git clone https://${USER}:${PASS}@github.com/org/repo.git
    '''
}
```

---

### Secret Text Example

Credential:

```text
api-token
```

Pipeline:

```groovy
withCredentials([
    string(
        credentialsId: 'api-token',
        variable: 'TOKEN'
    )
]) {

    sh '''
       curl \
       -H "Authorization: Bearer $TOKEN" \
       https://api.example.com
    '''
}
```

---

### SSH Key Example

Credential:

```text
github-ssh
```

```groovy
sshagent(['github-ssh']) {
    sh 'git clone git@github.com:org/repo.git'
}
```

---

### Secret File Example

```groovy
withCredentials([
    file(
        credentialsId: 'kubeconfig',
        variable: 'KUBECONFIG_FILE'
    )
]) {

    sh '''
       export KUBECONFIG=$KUBECONFIG_FILE
       kubectl get pods
    '''
}
```

---

### Inject Credentials as Environment Variables

```groovy
environment {
    API_TOKEN = credentials('api-token')
}
```

Usage:

```groovy
sh 'curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com'
```

---

### Best Practices

✅ Use credential IDs

✅ Use `withCredentials`

✅ Rotate secrets

✅ Use least privilege

❌ Never hardcode passwords

❌ Never print secrets in logs

---

# 7. Docker Integration

---

## Building Docker Images

```groovy
pipeline {

 agent any

 stages {

   stage('Build Image') {

      steps {
          sh '''
             docker build \
             -t app:${BUILD_NUMBER} .
          '''
      }
   }
 }
}
```

---

## Tagging Images

```groovy
sh '''
docker tag \
app:${BUILD_NUMBER} \
company/app:${BUILD_NUMBER}
'''
```

---

## Pushing Images

```groovy
withCredentials([
 usernamePassword(
   credentialsId: 'dockerhub',
   usernameVariable: 'USER',
   passwordVariable: 'PASS'
 )
]) {

 sh '''
 docker login -u $USER -p $PASS
 docker push company/app:${BUILD_NUMBER}
 '''
}
```

---

## Running Containers

```groovy
sh '''
docker run -d \
-p 8080:8080 \
app:${BUILD_NUMBER}
'''
```

---

## Docker Agent

```groovy
pipeline {

 agent {
    docker {
       image 'maven:3.9-eclipse-temurin-21'
    }
 }

 stages {

    stage('Build') {
       steps {
          sh 'mvn clean package'
       }
    }
 }
}
```

---

## Reusing Workspace

```groovy
agent {

 docker {

   image 'node:22'

   reuseNode true
 }
}
```

---

## Docker Compose Example

```groovy
pipeline {

 agent any

 stages {

   stage('Start Services') {
      steps {
         sh 'docker compose up -d'
      }
   }

   stage('Run Tests') {
      steps {
         sh 'npm test'
      }
   }

   stage('Cleanup') {
      steps {
         sh 'docker compose down'
      }
   }
 }
}
```

---

## Docker Buildx Example

```groovy
sh '''
docker buildx build \
--platform linux/amd64,linux/arm64 \
-t company/app:latest \
.
'''
```

---

## Publish Image to AWS ECR

```groovy
sh '''
aws ecr get-login-password \
| docker login \
--username AWS \
--password-stdin \
ACCOUNT.dkr.ecr.REGION.amazonaws.com

docker push ACCOUNT.dkr.ecr.REGION.amazonaws.com/app:latest
'''
```

---

# 8. Kubernetes Integration

---

## Jenkins Kubernetes Architecture

```mermaid
flowchart LR

 Controller[Jenkins Controller]

 Controller --> Plugin[Kubernetes Plugin]

 Plugin --> Pod1[Build Pod]
 Plugin --> Pod2[Test Pod]
 Plugin --> Pod3[Deploy Pod]

 Pod3 --> Cluster[Kubernetes Cluster]
```

---

## Kubernetes Plugin

Install:

```text
Kubernetes Plugin
```

---

## Kubernetes Cloud Configuration

```text
Manage Jenkins
→ Clouds
→ Kubernetes
```

Configure:

```text
Cluster URL
Namespace
Credentials
```

---

## Simple Kubernetes Agent

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
    image: maven:3.9.9-eclipse-temurin-21
    command:
    - cat
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

---

## Pod Template Example

```groovy
podTemplate(

 containers: [

   containerTemplate(
     name: 'node',
     image: 'node:22',
     ttyEnabled: true
   ),

   containerTemplate(
     name: 'docker',
     image: 'docker:dind',
     ttyEnabled: true
   )
 ]
) {

 node(POD_LABEL) {

   container('node') {

      sh 'npm install'
   }
 }
}
```

---

## Deploy Application

```groovy
stage('Deploy') {

 steps {

   sh '''
   kubectl apply \
   -f k8s/deployment.yaml
   '''
 }
}
```

---

## Deployment Status

```groovy
sh '''
kubectl rollout status \
deployment/my-app
'''
```

---

## Rolling Restart

```groovy
sh '''
kubectl rollout restart \
deployment/my-app
'''
```

---

## Rollback

```groovy
sh '''
kubectl rollout undo \
deployment/my-app
'''
```

---

## Blue-Green Deployment

```text
Blue Environment  -> Live
Green Environment -> New Version

Switch Traffic
```

Benefits:

* Zero downtime
* Fast rollback

---

## Canary Deployment

```text
10% Users -> New Version
90% Users -> Old Version
```

Gradually increase traffic.

---

# 9. CI/CD Examples

---

## Java Maven Pipeline

```groovy
pipeline {

 agent any

 stages {

   stage('Checkout') {
      steps {
         checkout scm
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
   }

   stage('Publish') {
      steps {
         archiveArtifacts '*.jar'
      }
   }
 }
}
```

---

## Gradle Pipeline

```groovy
pipeline {

 agent any

 stages {

   stage('Build') {

      steps {

         sh './gradlew build'
      }
   }
 }
}
```

---

## Node.js Pipeline

```groovy
pipeline {

 agent {
   docker {
      image 'node:22'
   }
 }

 stages {

   stage('Install') {
      steps {
         sh 'npm ci'
      }
   }

   stage('Test') {
      steps {
         sh 'npm test'
      }
   }

   stage('Build') {
      steps {
         sh 'npm run build'
      }
   }
 }
}
```

---

## Python Pipeline

```groovy
pipeline {

 agent any

 stages {

   stage('Install') {

      steps {
         sh '''
         pip install \
         -r requirements.txt
         '''
      }
   }

   stage('Test') {

      steps {
         sh 'pytest'
      }
   }
 }
}
```

---

## .NET Pipeline

```groovy
pipeline {

 agent any

 stages {

   stage('Restore') {
      steps {
         sh 'dotnet restore'
      }
   }

   stage('Build') {
      steps {
         sh 'dotnet build'
      }
   }

   stage('Test') {
      steps {
         sh 'dotnet test'
      }
   }
 }
}
```

---

## Complete CI/CD Example

```mermaid
flowchart LR

 Git[Git Push]
 --> Build[Build]

 Build
 --> Test[Test]

 Test
 --> Scan[Security Scan]

 Scan
 --> Package[Package]

 Package
 --> Docker[Docker Build]

 Docker
 --> Registry[Container Registry]

 Registry
 --> Deploy[Deploy]

 Deploy
 --> Verify[Smoke Test]
```

---

# 10. Useful Groovy Snippets

---

## Variables

```groovy
def version = "1.0.0"

echo version
```

---

## Read File

```groovy
def content = readFile('version.txt')

echo content
```

---

## Write File

```groovy
writeFile(
 file: 'build.txt',
 text: 'success'
)
```

---

## Check File Exists

```groovy
if (fileExists('pom.xml')) {

    echo "Maven Project"
}
```

---

## Loop Example

```groovy
for (int i = 0; i < 5; i++) {

    echo "Iteration ${i}"
}
```

---

## List Iteration

```groovy
def services = [
 "user",
 "order",
 "payment"
]

services.each {

    echo it
}
```

---

## Conditional Logic

```groovy
if (env.BRANCH_NAME == 'main') {

    echo 'Production Build'
}
else {

    echo 'Non Production'
}
```

---

## Try Catch

```groovy
try {

   sh 'mvn test'

}
catch(Exception ex) {

   echo ex.toString()

   currentBuild.result = 'FAILURE'
}
```

---

## JSON Parsing

```groovy
def json = readJSON(
 file: 'data.json'
)

echo json.name
```

---

## HTTP Request

```groovy
def response = httpRequest(
 url: 'https://api.example.com'
)

echo response.content
```

---

## Archive Artifacts

```groovy
archiveArtifacts(
 artifacts: '**/*.jar',
 fingerprint: true
)
```

---

## Shared Library Import

```groovy
@Library('shared-lib') _

buildApp()
```

---

## Dynamic Stage Generation

```groovy
def services = [
 'user',
 'order',
 'payment'
]

services.each { svc ->

 stage("Build-${svc}") {

   echo "Building ${svc}"
 }
}
```

---

## Current Build Information

```groovy
echo currentBuild.result

echo currentBuild.displayName

echo currentBuild.durationString
```

---

## Workspace Cleanup

```groovy
cleanWs()
```

Alternative:

```groovy
deleteDir()
```

---

## Sleep

```groovy
sleep(
 time: 10,
 unit: 'SECONDS'
)
```

---

## Retry

```groovy
retry(3) {

   sh './deploy.sh'
}
```

---

## Timeout

```groovy
timeout(
 time: 30,
 unit: 'MINUTES'
) {

   sh './long-running-task.sh'
}
```
