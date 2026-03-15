pipeline {
    agent any

    tools {
        maven 'maven3' // make sure Maven 3.9+ is installed in Jenkins
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    options {
        skipDefaultCheckout(true)
        // fail pipeline on any stage failure
        failFast true
    }

    stages {

        stage('Verify Tools') {
            steps {
                script {
                    // Detect JAVA_HOME dynamically for JDK 21
                    env.JAVA_HOME = sh(
                        script: "readlink -f \$(which javac) | sed 's:/bin/javac::'",
                        returnStdout: true
                    ).trim()
                    env.PATH = "${env.JAVA_HOME}/bin:${env.PATH}"
                }
                sh '''
                    echo "JAVA_HOME=${JAVA_HOME}"
                    java -version
                    mvn -version
                '''
            }
        }

        stage('Git Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: 'main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/fasil7170/Boardgame.git',
                        credentialsId: 'git-cred'
                    ]]
                ])
            }
        }

        stage('Compile') {
            steps {
                sh "mvn compile -Djava.version=21"
            }
        }

        stage('Test') {
            steps {
                sh "mvn test -Djava.version=21"
            }
        }

        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html . || true"
                archiveArtifacts artifacts: 'trivy-fs-report.html', allowEmptyArchive: true
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
                waitForQualityGate abortPipeline: true, credentialsId: 'sonar-token'
            }
        }

        stage('Package') {
            steps {
                sh "mvn package -Djava.version=21"
            }
        }

        stage('Publish To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', maven: 'maven3') {
                    sh "mvn deploy -Djava.version=21"
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                withDockerRegistry([credentialsId: 'docker-cred', url: '']) {
                    sh "docker build -t fazil2664/boardshack:latest ."
                    sh "docker push fazil2664/boardshack:latest"
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html fazil2664/boardshack:latest || true"
                archiveArtifacts artifacts: 'trivy-image-report.html', allowEmptyArchive: true
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'k8-cred', namespace: 'webapps') {
                    sh "kubectl apply -f deployment-service.yaml"
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished with status: ${currentBuild.currentResult}"
        }
    }
}
