pipeline {
    agent {
        kubernetes {
            label 'boardgame-agent'
            defaultContainer 'jnlp'
            yaml """
apiVersion: v1
kind: Pod
spec:
  imagePullSecrets:
    - name: local-registry-secret
  containers:
    - name: maven
      image: local-registry:5000/maven:3.9.4-eclipse-temurin-17
      imagePullPolicy: IfNotPresent
      command:
        - cat
      tty: true
    - name: docker
      image: local-registry:5000/docker:24.0.2-cli
      imagePullPolicy: IfNotPresent
      command:
        - cat
      tty: true
    - name: trivy
      image: local-registry:5000/aquasec/trivy:0.43.0
      imagePullPolicy: IfNotPresent
      command:
        - cat
      tty: true
    - name: sonar-scanner
      image: local-registry:5000/sonar-scanner-cli:5.13.0.40407
      imagePullPolicy: IfNotPresent
      command:
        - cat
      tty: true
"""
        }
    }

    environment {
        SCANNER_HOME = '/usr/local/sonar-scanner'
    }

    stages {

        stage('Git Checkout') {
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: 'main']],
                    userRemoteConfigs: [[url: 'https://github.com/fasil7170/Boardgame.git', credentialsId: 'git-cred']]
                ])
            }
        }

        stage('Compile') {
            steps {
                container('maven') {
                    sh "mvn compile"
                }
            }
        }

        stage('Test') {
            steps {
                container('maven') {
                    sh "mvn test"
                }
            }
        }

        stage('File System Scan') {
            steps {
                container('trivy') {
                    sh "trivy fs --format table -o trivy-fs-report.html . || true"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('sonar-scanner') {
                    withSonarQubeEnv('sonar') {
                        retry(3) {
                            sh """$SCANNER_HOME/bin/sonar-scanner \
                               -Dsonar.projectName=BoardGame \
                               -Dsonar.projectKey=BoardGame \
                               -Dsonar.java.binaries=. """
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Build') {
            steps {
                container('maven') {
                    sh "mvn package"
                }
            }
        }

        stage('Publish To Nexus') {
            steps {
                container('maven') {
                    withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', traceability: true) {
                        sh "mvn deploy"
                    }
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                container('docker') {
                    withDockerRegistry(credentialsId: 'docker-cred', url: 'https://local-registry:5000') {
                        sh "docker build -t local-registry:5000/fazil2664/boardshack:latest ."
                    }
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                container('trivy') {
                    sh "trivy image --format table -o trivy-image-report.html local-registry:5000/fazil2664/boardshack:latest || true"
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                container('docker') {
                    withDockerRegistry(credentialsId: 'docker-cred', url: 'https://local-registry:5000') {
                        sh "docker push local-registry:5000/fazil2664/boardshack:latest"
                    }
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'k8-cred', namespace: 'webapps') {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeConfig(credentialsId: 'k8-cred', namespace: 'webapps') {
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                }
            }
        }

    }

    post {
        always {
            script {
                echo "Pipeline finished with status: ${currentBuild.result ?: 'SUCCESS'}"
            }
        }
    }
}
