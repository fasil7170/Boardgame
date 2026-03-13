pipeline {
    agent {
        kubernetes {
            label 'jenkins-single-agent'
            defaultContainer 'jnlp'
            yaml """
apiVersion: v1
kind: Pod
spec:
  imagePullSecrets:
    - name: local-registry-secret   # replace with your registry secret if needed
  containers:
  - name: jnlp
    image: local-registry:5000/jenkins-agent:latest   # your custom prebuilt agent image
    imagePullPolicy: IfNotPresent
    tty: true
"""
        }
    }

    environment {
        SCANNER_HOME = "/opt/sonar-scanner"
        MAVEN_CMD = "mvn"
        DOCKER_IMAGE = "fazil2664/boardshack:latest"
        KUBE_NAMESPACE = "webapps"
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/fasil7170/Boardgame.git'
            }
        }

        stage('Compile') {
            steps {
                sh "${MAVEN_CMD} compile"
            }
        }

        stage('Test') {
            steps {
                sh "${MAVEN_CMD} test"
            }
        }

        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html . || true"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=BoardGame \
                        -Dsonar.projectKey=BoardGame \
                        -Dsonar.java.binaries=. """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
            }
        }

        stage('Build') {
            steps {
                sh "${MAVEN_CMD} package"
            }
        }

        stage('Publish to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', traceability: true) {
                    sh "${MAVEN_CMD} deploy"
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    sh "docker build -t ${DOCKER_IMAGE} ."
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html ${DOCKER_IMAGE} || true"
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    sh "docker push ${DOCKER_IMAGE}"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'k8-cred', namespace: "${KUBE_NAMESPACE}", serverUrl: 'https://192.168.0.100:6443') {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeConfig(credentialsId: 'k8-cred', namespace: "${KUBE_NAMESPACE}", serverUrl: 'https://192.168.0.100:6443') {
                    sh "kubectl get pods -n ${KUBE_NAMESPACE}"
                    sh "kubectl get svc -n ${KUBE_NAMESPACE}"
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished. You can add email notifications here if mail server is configured."
        }
    }
}
