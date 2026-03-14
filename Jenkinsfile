pipeline {
    agent any

    tools {
        // Use manually installed JDK and Maven
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = '/home/jenkins/agent/tools/sonar-scanner' // adjust if needed
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/fasil7170/Boardgame.git'
            }
        }

        stage('Compile') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'mvn compile'
                }
            }
        }

        stage('Test') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'mvn test'
                }
            }
        }

        stage('File System Scan') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'trivy fs --format table -o trivy-fs-report.html .'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withSonarQubeEnv('sonar') {
                        sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BoardGame -Dsonar.projectKey=BoardGame -Dsonar.java.binaries=."
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
                    sh 'mvn package'
                }
            }
        }

        stage('Publish To Nexus') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3') {
                        sh 'mvn deploy'
                    }
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    script {
                        withDockerRegistry(credentialsId: 'docker-cred', url: '', toolName: 'docker') {
                            sh 'docker build -t fazil2664/boardshack:latest .'
                        }
                    }
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'trivy image --format table -o trivy-image-report.html fazil2664/boardshack:latest'
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    script {
                        withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh 'docker push fazil2664/boardshack:latest'
                        }
                    }
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withKubeConfig(credentialsId: 'k8-cred', namespace: 'webapps', serverUrl: 'https://192.168.0.100:6443') {
                        sh 'kubectl apply -f deployment-service.yaml'
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withKubeConfig(credentialsId: 'k8-cred', namespace: 'webapps', serverUrl: 'https://192.168.0.100:6443') {
                        sh 'kubectl get pods -n webapps'
                        sh 'kubectl get svc -n webapps'
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                // Simplified email using sandbox-safe 'mail' step
                def status = currentBuild.currentResult
                mail to: 'rkf@gmail.com',
                     from: 'jenkins@example.com',
                     subject: "${env.JOB_NAME} - Build ${env.BUILD_NUMBER} - ${status}",
                     body: "Pipeline finished with status: ${status}\nCheck console output at ${env.BUILD_URL}"
            }
        }
    }
}
