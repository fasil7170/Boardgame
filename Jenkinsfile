pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main',
                credentialsId: 'git-cred',
                url: 'https://github.com/fasil7170/Boardgame.git'
            }
        }

        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Test') {
            steps {
                sh "mvn test"
            }
        }

        stage('File System Scan') {
            steps {
                sh "trivy fs --format html -o trivy-fs-report.html ."
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=BoardGame \
                    -Dsonar.projectKey=BoardGame \
                    -Dsonar.java.binaries=.
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false,
                    credentialsId: 'sonar-token'
                }
            }
        }

        stage('Build') {
            steps {
                sh "mvn package"
            }
        }

        stage('Publish To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings',
                          jdk: 'jdk17',
                          maven: 'maven3',
                          traceability: true) {
                    sh "mvn deploy"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t fazil2664/boardshack:latest ."
                    }
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format html -o trivy-image-report.html fazil2664/boardshack:latest"
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push fazil2664/boardshack:latest"
                    }
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8-cred',
                    namespace: 'webapps',
                    serverUrl: 'https://192.168.0.100:6443'
                ) {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8-cred',
                    namespace: 'webapps',
                    serverUrl: 'https://192.168.0.100:6443'
                ) {
                    sh "kubectl get pods"
                    sh "kubectl get svc"
                }
            }
        }
    }

    post {
        always {
            script {

                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus == 'SUCCESS' ? 'green' : 'red'

                def body = """
                <html>
                <body>
                <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                <h2>${jobName} - Build ${buildNumber}</h2>
                <div style="background-color: ${bannerColor}; padding: 10px;">
                <h3 style="color: white;">Pipeline Status: ${pipelineStatus}</h3>
                </div>
                <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                </div>
                </body>
                </html>
                """

                emailext (
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus}",
                    body: body,
                    to: 'rkf@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }
    }
}
