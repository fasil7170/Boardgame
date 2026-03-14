pipeline {
    agent any

    tools {
        jdk 'jdk17'        // Use manually installed JDK
        maven 'maven3'     // Use manually installed Maven
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Git Checkout') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    git branch: 'main',
                        credentialsId: 'git-cred',
                        url: 'https://github.com/fasil7170/Boardgame.git'
                }
            }
        }

        stage('Compile') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh "mvn compile"
                }
            }
        }

        stage('Test') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh "mvn test"
                }
            }
        }

        stage('File System Scan') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh "trivy fs --format html -o trivy-fs-report.html ."
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
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
        }

        stage('Quality Gate') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    script {
                        waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                    }
                }
            }
        }

        stage('Build') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh "mvn package"
                }
            }
        }

        stage('Publish To Nexus') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', traceability: true) {
                        sh "mvn deploy"
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    script {
                        withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh "docker build -t fazil2664/boardshack:latest ."
                        }
                    }
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh "trivy image --format html -o trivy-image-report.html fazil2664/boardshack:latest"
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    script {
                        withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh "docker push fazil2664/boardshack:latest"
                        }
                    }
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withKubeConfig(
                        credentialsId: 'k8-cred',
                        namespace: 'webapps',
                        serverUrl: 'https://127.0.0.1:6443' // Local cluster
                    ) {
                        sh "kubectl apply -f deployment-service.yaml"
                        sh "kubectl rollout status deployment/boardgame"
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withKubeConfig(
                        credentialsId: 'k8-cred',
                        namespace: 'webapps',
                        serverUrl: 'https://127.0.0.1:6443'
                    ) {
                        sh "kubectl get all -n webapps"
                    }
                }
            }
        }
    }

    post {
        always {
            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                emailext(
                    subject: "${env.JOB_NAME} - Build ${env.BUILD_NUMBER} - ${currentBuild.result ?: 'UNKNOWN'}",
                    body: """
                        <html>
                        <body>
                        <h2>${env.JOB_NAME} - Build ${env.BUILD_NUMBER}</h2>
                        <p>Pipeline Status: ${currentBuild.result ?: 'UNKNOWN'}</p>
                        <p>Check the <a href="${env.BUILD_URL}">console output</a>.</p>
                        </body>
                        </html>
                    """,
                    to: 'rkf@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }
    }
}
