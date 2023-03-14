### Complete CI/CD Pipeline with EKS and private DockerHub registry

### Technologies used:

Kubernetes, Jenkins, AWS EKS, Docker Hub, Java, Maven, Linux, Docker,Git

### Project Description:

1- Write K8s manifest files for Deployment and Service configuration

2- Integrate deploy step in the CI/CD pipeline to deploy newly built application image from DockerHub private registry to the EKS cluster

3-So the complete CI/CD project we build has the following configuration:

   a. CI step: Increment version
   
   b. CI step: Build artifact for Java Maven application
   
   c. CI step: Build and push Docker image to DockerHub
   
   d. CD step: Deploy new application version to EKS cluster e. CD step: Commit the version update

### Instructions:

###### Step 1: Install and configure Jenkins and docker on DigitalOcean Droplet Server

###### Step 2: Create AWS EKS cluster with Nodegroups on AWS

###### Step 3: Install gettext-base tool for envsubst on jenkins container server

```
apt-get update
```

```
apt-get install gettext-base
```

###### Step 4: Create Java app deployment and service component config file

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-maven-app
  labels:
    app: java-maven-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: java-maven-app
  template:
    metadata:
      labels:
        app: java-maven-app
    spec:
      imagePullSecrets:
        - name: my-registry-key
      containers:
      - name: java-maven-app
        image: jason8746/java-app:$IMAGE_NAME
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
```

```
apiVersion: v1
kind: Service
metadata:
  name: java-maven-app
spec:
  selector:
    app: java-maven-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

###### Step 5: Update the Deploy stage in Jenkinsfile

```

pipeline {
    agent any
    tools {
        maven 'Maven'
    }
    stages {
        stage('increment version') {
            steps {
                script {
                    echo 'incrementing app version...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    def version = matcher[0][1]
                    env.IMAGE_NAME = "$version-$BUILD_NUMBER"
                }
            }
        }
        stage('build app') {
            steps {
                script {
                    echo "building the application..."
                    sh 'mvn clean package'
                }
            }
        }
        stage('build image') {
            steps {
                script {
                    echo "building the docker image..."
                    withCredentials([usernamePassword(credentialsId: 'docker-hub', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh "docker build -t nanajanashia/demo-app:${IMAGE_NAME} ."
                        sh "echo $PASS | docker login -u $USER --password-stdin"
                        sh "docker push nanajanashia/demo-app:${IMAGE_NAME}"
                    }
                }
            }
        }
        stage('deploy') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
                APP_NAME = 'java-maven-app'
            }

            steps {
                script {

                  withCredentials([usernamePassword(credentialsId: 'docker-hub', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                      sh "kubectl create secret docker-registry my-registry-key --docker-server=docker.io --docker-username=$USER --docker-password=$PASS"
                  }
                  sh 'envsubst < kubernetes/deployment.yaml | kubectl apply -f -'
                  sh 'envsubst < kubernetes/service.yaml | kubectl apply -f -'

                }
            }
        }
        stage('commit version update') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'gitlab-credentials', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        // git config here for the first time run
                        sh 'git config --global user.email "jenkins@example.com"'
                        sh 'git config --global user.name "jenkins"'

                        sh "git remote set-url origin https://${USER}:${PASS}@gitlab.com/nanuchi/java-maven-app.git"
                        sh 'git add .'
                        sh 'git commit -m "ci: version bump"'
                        sh 'git push origin HEAD:jenkins-jobs'
                    }
                }
            }
        }
    }
}
```
