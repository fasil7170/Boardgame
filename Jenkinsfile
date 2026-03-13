pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        MAVEN_OPTS = "-Xmx1024m -XX:+UseG1GC"
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
                sh "mvn compile -Dmaven.compiler.source=17 -Dmaven.compiler.target=17"
            }
        }

        // =========================
        // 3. Run Unit Tests (Parallel & Optimized)
        // =========================
        stage('Test') {
            steps {
                sh """
                mvn test \
                    -B \
                    -T 2C \
                    -Dspring.main.lazy-initialization=true \
                    -Dspring.main.web-application-type=none \
                    -Dspring.jpa.open-in-view=false
                """
            }
        }

        // =========================
        // 4 & 10. Security Scans in Parallel
        // =========================
        stage('Security Scans') {
            parallel {
                stage('File System Scan') {
                    steps {
                        sh "trivy fs --format table -o trivy-fs-report.html ."
                    }
                }
                stage('Docker Image Scan') {
                    steps {
                        script {
                            sh "docker build -t fazil2664/boardshack:latest ."
                            sh "trivy image --format table -o trivy-image-report.html fazil2664/boardshack:latest"
                        }
                    }
                }
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

    post {
        always {
            script {
                try {
                    mail(
                        to: 'rkf@gmail.com',
                        subject: "${env.JOB_NAME} - Build ${env.BUILD_NUMBER} - ${currentBuild.currentResult}",
                        body: "Pipeline Status: ${currentBuild.currentResult}\nBuild URL: ${env.BUILD_URL}"
                    )
                } catch (Exception e) {
                    echo "Email skipped: ${e.message}"
                }
            }
        }
    }
}
