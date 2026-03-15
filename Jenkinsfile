pipeline {
    agent any

    tools {
        maven 'maven3'  // Ensure this matches your Jenkins tool name
    }

    environment {
        // Detect JAVA_HOME dynamically for the agent
        JAVA_HOME = ''
        PATH = "${env.JAVA_HOME}/bin:${env.PATH}"
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Verify Tools') {
            steps {
                script {
                    // Dynamically detect JAVA_HOME
                    env.JAVA_HOME = sh(script: "readlink -f \$(which javac) | sed 's:/bin/javac::'", returnStdout: true).trim()
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
                sh "trivy fs --format table -o trivy-fs-report.html ."
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
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
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
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: '', maven: 'maven3', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }

        stage('Build & Tag Docker Image') {
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
                sh "trivy image --format table -o trivy-image-report.html fazil2664/boardshack:latest"
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
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
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
