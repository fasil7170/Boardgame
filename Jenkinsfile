pipeline {

    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest
    tty: true

  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
    command:
    - cat
    tty: true
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker

  volumes:
  - name: docker-config
    secret:
      secretName: dockerhub-secret
"""
        }
    }

    environment {
        JAVA_HOME = "/usr/lib/jvm/java-17-openjdk-amd64"
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_NAME = "fazil2664/boardshack:latest"
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main',
                credentialsId: 'git-cred',
                url: 'https://github.com/fasil7170/Boardgame.git'
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
                    ${SCANNER_HOME}/bin/sonar-scanner \
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
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Build Artifact') {
            steps {
                sh "mvn package -DskipTests"
            }
        }

        stage('Publish To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings') {
                    sh "mvn deploy"
                }
            }
        }

        stage('Build & Push Image (Kaniko)') {
            steps {
                container('kaniko') {
                    sh """
                    /kaniko/executor \
                      --context `pwd` \
                      --dockerfile Dockerfile \
                      --destination $IMAGE_NAME \
                      --skip-tls-verify
                    """
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
                    sh "kubectl get pods"
                    sh "kubectl get svc"
                }
            }
        }
    }

    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'SUCCESS'

                def body = """
                Job: ${jobName}
                Build: ${buildNumber}
                Status: ${pipelineStatus}
                URL: ${env.BUILD_URL}
                """

                try {
                    emailext(
                        subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus}",
                        body: body,
                        to: "rkf@gmail.com"
                    )
                } catch (Exception e) {
                    echo "Email notification failed: ${e}"
                }
            }
        }
    }
}
