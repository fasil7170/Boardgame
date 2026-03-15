pipeline {
    agent any

    tools {
        maven 'maven3' // ensure Maven 3.9+ installed in Jenkins
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    options {
        skipDefaultCheckout(true)
    }

    stages {
        stage('Verify Tools') {
            steps {
                script {
                    // Dynamically detect JAVA_HOME for the agent
                    env.JAVA_HOME = sh(
                        script: "readlink -f \$(which javac) | sed 's:/bin/javac::'",
                        returnStdout: true
                    ).trim()
                    env.PATH = "${env.JAVA_HOME}/bin:${env.PATH}"
                }
                sh '''
                    echo "JAVA_HOME=${JAVA_HOME}"
                    java -version
                    mvn -version
                '''
            }
        }

        stage('Git Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: 'main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/fasil7170/Boardgame.git',
                        credentialsId: 'git-cred'
                    ]]
                ])
            }
        }

        stage('Compile') {
            steps {
                script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        sh "mvn compile -Djava.version=21"
                    }
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        sh "mvn test -Djava.version=21"
                    }
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

        stage('Package') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh "mvn package -Djava.version=21"
                }
            }
        }

        stage('Publish To Nexus') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withMaven(globalMavenSettingsConfig: 'global-settings', maven: 'maven3') {
                        sh "mvn deploy -Djava.version=21"
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh "docker build -t fazil2664/boardshack:latest ."
                            sh "docker push fazil2664/boardshack:latest"
                        }
                    }
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withKubeConfig(credentialsId: 'k8-cred', namespace: 'webapps') {
                        sh "kubectl apply -f deployment-service.yaml"
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
