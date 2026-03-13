pipeline {
    agent {
        kubernetes {
            label 'jenkins-agent'
            defaultContainer 'jnlp'
            yaml """
apiVersion: v1
kind: Pod
metadata:
  name: jenkins-agent
spec:
  imagePullSecrets:
  - name: dockerhub-secret   # optional, for Docker Hub auth if rate-limited
  containers:
  - name: maven
    image: maven:3.9.4-eclipse-temurin-17
    command:
    - cat
    tty: true
  - name: docker
    image: docker:24.0.2-cli
    command:
    - cat
    tty: true
  - name: trivy
    image: aquasec/trivy:0.43.0
    command:
    - cat
    tty: true
  - name: sonar-scanner
    image: sonarsource/sonar-scanner-cli:5.13.0.40407
    command:
    - cat
    tty: true
"""
        }
    }

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = '/opt/sonar-scanner'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/fasil7170/Boardgame.git'
            }
        }

        stage('Build & Test Parallel') {
            parallel {
                stage('Compile & Test') {
                    steps {
                        container('maven') {
                            sh "mvn clean compile test"
                        }
                    }
                }

                stage('File System Scan') {
                    steps {
                        container('trivy') {
                            sh "trivy fs --format table -o trivy-fs-report.html ."
                        }
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('sonar-scanner') {
                    withSonarQubeEnv('sonar') {
                        sh """$SCANNER_HOME/bin/sonar-scanner \
                             -Dsonar.projectName=BoardGame \
                             -Dsonar.projectKey=BoardGame \
                             -Dsonar.java.binaries=. """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false
            }
        }

        stage('Package & Publish Parallel') {
            parallel {
                stage('Build & Publish To Nexus') {
                    steps {
                        container('maven') {
                            withMaven(globalMavenSettingsConfig: 'global-settings') {
                                sh "mvn package deploy"
                            }
                        }
                    }
                }

                stage('Docker Build, Scan & Push') {
                    steps {
                        container('docker') {
                            script {
                                def imageTag = "fazil2664/boardshack:${env.BUILD_NUMBER}"
                                
                                withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                                    sh "docker build -t ${imageTag} ."
                                    sh "docker push ${imageTag}"
                                    sh "docker tag ${imageTag} fazil2664/boardshack:latest"
                                    sh "docker push fazil2664/boardshack:latest"
                                }
                            }
                        }
                        container('trivy') {
                            sh "trivy image --format table -o trivy-image-report.html fazil2664/boardshack:${env.BUILD_NUMBER}"
                        }
                    }
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'k8-cred', namespace: 'webapps') {
                    sh "kubectl apply -f deployment-service.yaml"
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                }
            }
        }
    }

    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'SUCCESS'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

                def body = """
<html>
<body>
<div style="border: 4px solid ${bannerColor}; padding: 10px;">
<h2>${jobName} - Build ${buildNumber}</h2>
<div style="background-color: ${bannerColor}; padding: 10px;">
<h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
</div>
<p>Check the console output at <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
</div>
</body>
</html>
"""

                mail to: 'rkf@gmail.com',
                     subject: "${jobName} - Build #${buildNumber} - ${pipelineStatus.toUpperCase()}",
                     body: body
            }
        }
    }
}
