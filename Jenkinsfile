pipeline {
    agent {
        kubernetes {
            inheritFrom 'default'
            defaultContainer 'jnlp'
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-agent
spec:
  imagePullSecrets:
    - name: dockerhub-secret  # optional for private images
  containers:
    - name: maven
      image: maven:3.9.4-eclipse-temurin-17
      imagePullPolicy: IfNotPresent
      command: ["cat"]
      tty: true
    - name: docker
      image: docker:24.0.2-cli
      imagePullPolicy: IfNotPresent
      command: ["cat"]
      tty: true
    - name: trivy
      image: aquasec/trivy:0.43.0
      imagePullPolicy: IfNotPresent
      command: ["cat"]
      tty: true
    - name: sonar-scanner
      image: sonarsource/sonar-scanner-cli:5.14.0.61720
      imagePullPolicy: IfNotPresent
      command: ["cat"]
      tty: true
"""
        }
    }

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = '/opt/sonar-scanner'
    }

    options {
        timeout(time: 60, unit: 'MINUTES')
        retry(2)  // retry pipeline for transient image/pod errors
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/fasil7170/Boardgame.git'
            }
        }

        stage('Compile & Test') {
            parallel {
                stage('Compile & Unit Test') {
                    steps {
                        container('maven') {
                            sh "mvn clean compile test"
                        }
                    }
                }
                stage('Trivy FS Scan') {
                    steps {
                        container('trivy') {
                            sh "trivy fs --format table -o trivy-fs-report.html . || echo 'Trivy FS scan failed'"
                        }
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('sonar-scanner') {
                    withSonarQubeEnv('sonar') {
                        sh """$SCANNER_HOME/bin/sonar-scanner \
                             -Dsonar.projectName=BoardGame \
                             -Dsonar.projectKey=BoardGame \
                             -Dsonar.java.binaries=. """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false
            }
        }

        stage('Build & Deploy to Nexus') {
            steps {
                container('maven') {
                    withMaven(globalMavenSettingsConfig: 'global-settings') {
                        sh "mvn package deploy"
                    }
                }
            }
        }

        stage('Docker Build, Scan & Push') {
            steps {
                container('docker') {
                    script {
                        def imageTag = "fazil2664/boardshack:${env.BUILD_NUMBER}"

                        withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                            sh "docker build -t ${imageTag} . || echo 'Docker build failed'"
                            sh "docker push ${imageTag} || echo 'Docker push failed'"
                            sh "docker tag ${imageTag} fazil2664/boardshack:latest || true"
                            sh "docker push fazil2664/boardshack:latest || true"
                        }
                    }
                }

                container('trivy') {
                    sh "trivy image --format table -o trivy-image-report.html fazil2664/boardshack:${env.BUILD_NUMBER} || echo 'Trivy image scan failed'"
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'k8-cred', namespace: 'webapps') {
                    sh "kubectl apply -f deployment-service.yaml"
                    sh "kubectl rollout status deployment/boardgame-deployment -n webapps"
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                }
            }
        }
    }

    post {
        always {
            script {
                echo "Pipeline finished. Status: ${currentBuild.currentResult}"
                // Optional email step (requires valid SMTP)
                // mail(to: 'rkf@gmail.com', subject: "Build ${currentBuild.fullDisplayName}", body: "Status: ${currentBuild.currentResult}")
            }
        }
    }
}
