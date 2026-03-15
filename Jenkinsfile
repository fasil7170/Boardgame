pipeline {
    agent any

    tools {
        maven 'maven3'
        jdk 'jdk-21' // Make sure Jenkins has a JDK 21 tool configured
    }

    environment {
        PATH = "${tool 'jdk-21'}/bin:${env.PATH}"
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Verify Tools') {
            steps {
                sh '''
                    echo "JAVA_HOME=${JAVA_HOME}"
                    java -version
                    mvn -version
                '''
            }
        }

        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/fasil7170/Boardgame.git'
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
                    sh "trivy fs --format table -o trivy-fs-report.html ."
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
                    withMaven(globalMavenSettingsConfig: 'global-settings', maven: 'maven3') {
                        sh "mvn deploy"
                    }
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t fazil2664/boardshack:latest ."
                    }
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh "trivy image --format table -o trivy-image-report.html fazil2664/boardshack:latest"
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push fazil2664/boardshack:latest"
                    }
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withKubeConfig(credentialsId: 'k8-cred', namespace: 'webapps', serverUrl: 'https://192.168.0.100:6443') {
                        sh "kubectl apply -f deployment-service.yaml"
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withKubeConfig(credentialsId: 'k8-cred', namespace: 'webapps', serverUrl: 'https://192.168.0.100:6443') {
                        sh "kubectl get pods -n webapps"
                        sh "kubectl get svc -n webapps"
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished with status: ${currentBuild.currentResult}"
        }
    }
}
