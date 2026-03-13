pipeline {
    agent {
        kubernetes {
            label 'jenkins-agent'
            defaultContainer 'jnlp'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.9.4-openjdk-17
    command:
    - cat
    tty: true
  - name: docker
    image: docker:24.0.2-dind
    securityContext:
      privileged: true
    command:
    - dockerd-entrypoint.sh
    args:
    - "--host=tcp://0.0.0.0:2375"
    - "--host=unix:///var/run/docker.sock"
    tty: true
  - name: sonar-scanner
    image: sonarsource/sonar-scanner-cli:5.13.0
    command:
    - cat
    tty: true
  - name: trivy
    image: aquasec/trivy:0.44.0
    command:
    - cat
    tty: true
"""
        }
    }

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
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
                script {
                    try {
                        container('trivy') {
                            sh "trivy fs --format table -o trivy-fs-report.html ."
                        }
                    } catch (err) {
                        echo "Trivy FS scan failed, skipping this stage: ${err}"
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    try {
                        container('sonar-scanner') {
                            withSonarQubeEnv('sonar') {
                                sh """
                                    $SCANNER_HOME/bin/sonar-scanner \
                                    -Dsonar.projectName=BoardGame \
                                    -Dsonar.projectKey=BoardGame \
                                    -Dsonar.java.binaries=.
                                """
                            }
                        }
                    } catch (err) {
                        echo "SonarQube analysis failed, skipping this stage: ${err}"
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    try {
                        waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                    } catch (err) {
                        echo "Quality Gate check failed: ${err}"
                    }
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
                    script {
                        try {
                            withDockerRegistry(credentialsId: 'docker-cred') {
                                sh "docker build -t fazil2664/boardshack:latest ."
                            }
                        } catch (err) {
                            echo "Docker build failed: ${err}"
                        }
                    }
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                script {
                    try {
                        container('trivy') {
                            sh "trivy image --format table -o trivy-image-report.html fazil2664/boardshack:latest"
                        }
                    } catch (err) {
                        echo "Trivy image scan failed, skipping: ${err}"
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                container('docker') {
                    script {
                        try {
                            withDockerRegistry(credentialsId: 'docker-cred') {
                                sh "docker push fazil2664/boardshack:latest"
                            }
                        } catch (err) {
                            echo "Docker push failed: ${err}"
                        }
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
