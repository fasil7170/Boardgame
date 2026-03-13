pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_NAME = "fazil2664/boardshack"
    }

    stages {

        // =========================
        // 1. Checkout Code
        // =========================
        stage('Git Checkout') {
            steps {
                git branch: 'main',
                credentialsId: 'git-cred',
                url: 'https://github.com/fasil7170/Boardgame.git'
            }
        }

        // =========================
        // 2. Build + Fast Tests
        // =========================
        stage('Build & Fast Test') {
            steps {
                echo "Running optimized build and tests..."
                sh """
                mvn -B clean package \
                -Dspring.main.web-application-type=none \
                -Dspring.main.lazy-initialization=true \
                -DskipITs
                """
            }
        }

        // =========================
        // 3. File System Scan
        // =========================
        stage('File System Scan') {
            steps {
                sh "trivy fs --format html -o trivy-fs-report.html ."
            }
        }

        // =========================
        // 4. SonarQube Analysis
        // =========================
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=BoardGame \
                    -Dsonar.projectKey=BoardGame \
                    -Dsonar.java.binaries=target/classes
                    """
                }
            }
        }

        // =========================
        // 5. SonarQube Quality Gate
        // =========================
        stage('Quality Gate') {
            steps {
                script {
                    def qg = waitForQualityGate()
                    if (qg.status != 'OK') {
                        error "Pipeline aborted due to quality gate failure: ${qg.status}"
                    }
                }
            }
        }

        // =========================
        // 6. Publish to Nexus
        // =========================
        stage('Publish To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3') {
                    sh "mvn deploy -DskipTests"
                }
            }
        }

        // =========================
        // 7. Build Docker Image
        // =========================
        stage('Build Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t $IMAGE_NAME:latest ."
                        sh "docker tag $IMAGE_NAME:latest $IMAGE_NAME:${BUILD_NUMBER}"
                    }
                }
            }
        }

        // =========================
        // 8. Docker Image Scan
        // =========================
        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table $IMAGE_NAME:latest"
            }
        }

        // =========================
        // 9. Push Docker Image
        // =========================
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push $IMAGE_NAME:latest"
                        sh "docker push $IMAGE_NAME:${BUILD_NUMBER}"
                    }
                }
            }
        }

        // =========================
        // 10. Deploy to Kubernetes
        // =========================
        stage('Deploy To Kubernetes') {
            steps {
                script {
                    withKubeConfig(
                        credentialsId: 'k8-cred',
                        namespace: 'webapps',
                        serverUrl: 'https://192.168.0.100:6443'
                    ) {
                        try {
                            sh "kubectl apply -f deployment-service.yaml"
                            sh "kubectl rollout status deployment/boardgame-deployment -n webapps"
                        } catch (Exception e) {
                            echo "Deployment failed! Rolling back..."
                            sh "kubectl rollout undo deployment/boardgame-deployment -n webapps"
                            error "Deployment failed and rollback executed"
                        }
                    }
                }
            }
        }

        // =========================
        // 11. Verify Deployment
        // =========================
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

    // =========================
    // Post Actions
    // =========================
    post {
        always {
            echo "Pipeline completed: ${currentBuild.currentResult}"
            cleanWs()
        }
    }
}
