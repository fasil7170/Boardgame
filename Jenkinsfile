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
        // =========================
        // 1. Checkout from Git
        // =========================
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/fasil7170/Boardgame.git'
            }
        }
        
        // =========================
        // 2. Compile
        // =========================
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        // =========================
        // 3. Unit Tests
        // =========================
        stage('Test') {
            steps {
                sh "mvn test"
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
        // 12. Deploy to Kubernetes
        // =========================
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
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

                def body = """
                    <html>
                    <body>
                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding: 10px;">
                    <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                    </div>
                    </body>
                    </html>
                """

                emailext (
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'rkf@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }
    }
}
