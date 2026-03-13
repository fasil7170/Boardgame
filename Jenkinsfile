pipeline {
    agent any

    tools {
        jdk 'jdk17'        // Ensure JDK 17 is installed in Jenkins
        maven 'maven3'     // Ensure Maven 3 is installed in Jenkins
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        MAVEN_OPTS = "-Xmx1024m"   // Limit Maven memory usage
    }

    stages {
        // =========================
        // 1. Checkout from Git
        // =========================
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/fasil7170/Boardgame.git'
            }
        }

        // =========================
        // 2. Compile with Java 17
        // =========================
        stage('Compile') {
            steps {
                sh "mvn clean compile -Dmaven.compiler.source=17 -Dmaven.compiler.target=17"
            }
        }

        // =========================
        // 3. Run Unit Tests (optimized)
        // =========================
        stage('Test') {
            steps {
                echo "Running optimized Spring Boot tests"
                sh """
                mvn -B test \
                -Dspring.main.lazy-initialization=true \
                -Dspring.main.web-application-type=none \
                -Dspring.jpa.open-in-view=false
                """
            }
        }

        // =========================
        // 4. File System Security Scan
        // =========================
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        // =========================
        // 5. SonarQube Analysis
        // =========================
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=BoardGame \
                        -Dsonar.projectKey=BoardGame \
                        -Dsonar.java.binaries=target/classes"""
                }
            }
        }

        // =========================
        // 6. SonarQube Quality Gate
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
        // 7. Build Package
        // =========================
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }

        // =========================
        // 8. Deploy to Nexus
        // =========================
        stage('Publish To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }

        // =========================
        // 9. Build & Tag Docker Image
        // =========================
        stage('Build Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t fazil2664/boardshack:latest ."
                        sh "docker tag fazil2664/boardshack:latest fazil2664/boardshack:${env.BUILD_NUMBER}"
                    }
                }
            }
        }

        // =========================
        // 10. Docker Image Security Scan
        // =========================
        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html fazil2664/boardshack:latest"
            }
        }

        // =========================
        // 11. Push Docker Image
        // =========================
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push fazil2664/boardshack:latest"
                        sh "docker push fazil2664/boardshack:${env.BUILD_NUMBER}"
                    }
                }
            }
        }

        // =========================
        // 12. Deploy to Kubernetes with Rollback
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
                            error "Deployment failed and rollback executed: ${e.message}"
                        }
                    }
                }
            }
        }

        // =========================
        // 13. Verify Deployment
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
    // Post Actions: Email Notification
    // =========================
    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'

                // Make sure your SMTP server is configured or comment this out
                try {
                    mail(
                        to: 'rkf@gmail.com',
                        subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                        body: """
Pipeline Status: ${pipelineStatus}
Build URL: ${env.BUILD_URL}
"""
                    )
                } catch (Exception e) {
                    echo "Mail failed: ${e.message}"
                }
            }
        }
    }
}
