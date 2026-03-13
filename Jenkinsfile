pipeline {
    agent {
        kubernetes {
            label 'boardgame-agent'
            defaultContainer 'maven'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.9.2-jdk-17
    command:
    - cat
    tty: true
  - name: docker
    image: docker:24.0.5
    command:
    - cat
    tty: true
"""
        }
    }

    environment {
        SCANNER_HOME = '/tmp/sonar-scanner'
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/fasil7170/Boardgame.git'
            }
        }

        stage('Compile') {
            steps {
                container('maven') {
                    sh 'mvn compile'
                }
            }
        }

        stage('Test') {
            steps {
                container('maven') {
                    sh 'mvn test'
                }
            }
        }

        stage('Trivy File Scan') {
            steps {
                container('maven') {
                    sh '''
                        curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh
                        ./trivy fs --format table -o trivy-fs-report.html .
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('maven') {
                    sh '''
                        curl -L -o sonar-scanner.zip https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.14.0.61720-linux.zip
                        unzip sonar-scanner.zip -d /tmp/
                        export PATH=$SCANNER_HOME/bin:$PATH
                        /tmp/sonar-scanner-5.14.0.61720-linux/bin/sonar-scanner \
                            -Dsonar.projectKey=BoardGame \
                            -Dsonar.projectName=BoardGame \
                            -Dsonar.java.binaries=.
                    '''
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

        stage('Build & Package') {
            steps {
                container('maven') {
                    sh 'mvn package'
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                container('docker') {
                    withDockerRegistry(credentialsId: 'docker-cred', url: '') {
                        sh '''
                            docker build -t fazil2664/boardshack:latest .
                            docker push fazil2664/boardshack:latest
                        '''
                    }
                }
            }
        }

        stage('Trivy Docker Scan') {
            steps {
                container('docker') {
                    sh '''
                        curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh
                        ./trivy image --format table -o trivy-image-report.html fazil2664/boardshack:latest
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('docker') {
                    withKubeConfig(credentialsId: 'k8-cred', namespace: 'webapps') {
                        sh 'kubectl apply -f deployment-service.yaml'
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
                def status = currentBuild.currentResult
                def body = """
                <html>
                <body>
                <h2>BoardGame Pipeline - Build #${env.BUILD_NUMBER}</h2>
                <p>Status: ${status}</p>
                <p>Console Output: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                </body>
                </html>
                """
                emailext (
                    subject: "BoardGame Build #${env.BUILD_NUMBER} - ${status}",
                    body: body,
                    to: 'rkf@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }
    }
}
