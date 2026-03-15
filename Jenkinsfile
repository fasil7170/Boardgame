pipeline {
    agent any

    tools {
        maven 'maven3'  // Use the Maven installation configured in Jenkins
    }

    environment {
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk'   // Ensure this path exists on your agent
        PATH = "${env.JAVA_HOME}/bin:${env.PATH}"
        SCANNER_HOME = '/home/jenkins/agent/tools/sonar-scanner'  // Adjust if needed
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/fasil7170/Boardgame.git'
            }
        }

        stage('Verify Tools') {
            steps {
                sh '''
                    java -version
                    mvn -version
                    docker --version || echo "Docker not installed"
                '''
            }
        }

        stage('Compile') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'mvn clean compile'
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
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Build Artifact') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'mvn package'
                }
            }
        }

        stage('Publish To Nexus') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withMaven(globalMavenSettingsConfig: 'global-settings') {
                        sh 'mvn deploy'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    script {
                        withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
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
            echo "Pipeline completed with status: ${currentBuild.currentResult ?: 'UNKNOWN'}"
        }
    }
}
