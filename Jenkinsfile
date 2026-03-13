pipeline {
    agent {
        kubernetes {
            label 'jenkins-agent'
            defaultContainer 'jnlp'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.9.4-openjdk-17
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
    image: sonarsource/sonar-scanner-cli:5.12.0
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

        stage('Package & Publish') {
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

                stage('Docker Build & Scan') {
                    steps {
                        container('docker') {
                            withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                                sh "docker build -t fazil2664/boardshack:latest ."
                            }
                        }
                        container('trivy') {
                            sh "trivy image --format table -o trivy-image-report.html fazil2664/boardshack:latest"
                        }
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                container('docker') {
                    withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                        sh "docker push fazil2664/boardshack:latest"
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
<p>Check the <a href="${BUILD_URL}">console output</a>.</p>
</div>
</body>
</html>
"""

                emailext (
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'rkf@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }
    }
}
