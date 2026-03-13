pipeline {
    agent {
        kubernetes {
            label 'boardgame-single-agent'
            defaultContainer 'agent'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: agent
    image: myregistry/my-jenkins-agent:latest
    command:
    - cat
    tty: true
"""
        }
    }

    environment {
        SCANNER_HOME = "/opt/sonar-scanner" // adjust if your image has Sonar Scanner elsewhere
        DOCKER_CLI = "/usr/bin/docker"      // adjust if needed
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/fasil7170/Boardgame.git'
            }
        }

        stage('Compile') {
            steps {
                sh "mvn clean compile"
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
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    sh "$DOCKER_CLI build -t fazil2664/boardshack:latest ."
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
                    sh "$DOCKER_CLI login -u USER -p PASS" // Or use withCredentials if secret
                    sh "$DOCKER_CLI push fazil2664/boardshack:latest"
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'k8-cred', namespace: 'webapps', serverUrl: 'https://192.168.0.100:6443') {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeConfig(credentialsId: 'k8-cred', namespace: 'webapps', serverUrl: 'https://192.168.0.100:6443') {
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                }
            }
        }
    }

    post {
        always {
            script {
                def status = currentBuild.result ?: 'UNKNOWN'
                echo "Pipeline finished with status: ${status}"
            }
        }
    }
}
