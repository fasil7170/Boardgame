pipeline {
    agent any

    tools {
        // Ensure Maven 3+ is installed in Jenkins Global Tool Config
        maven 'maven3'
    }

    options {
        // We'll do Git checkout manually
        skipDefaultCheckout(true)
    }

    environment {
        // Sonar scanner installed in Jenkins Global Tool Config
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Verify Tools') {
            steps {
                script {
                    // Dynamically detect JAVA_HOME for JDK 21
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
                script {
                    // Stop pipeline if compile fails
                    sh "mvn compile"
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    // Stop pipeline if tests fail
                    sh "mvn test"
                }
            }
        }

        stage('Package') {
            steps {
                script {
                    sh "mvn package"
                }
            }
        }

        stage('Publish To Nexus') {
            steps {
                script {
                    withMaven(globalMavenSettingsConfig: 'global-settings', maven: 'maven3') {
                        sh "mvn deploy"
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    // Docker registry URL provided
                    withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                        sh "docker build -t fazil2664/boardshack:latest ."
                        sh "docker push fazil2664/boardshack:latest"
                    }
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                script {
                    withKubeConfig(credentialsId: 'k8-cred', namespace: 'webapps') {
                        sh "kubectl apply -f deployment-service.yaml"
                        sh "kubectl get pods -n webapps"
                        sh "kubectl get svc -n webapps"
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
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
        }

        stage('Quality Gate') {
            steps {
                script {
                    // Abort pipeline if quality gate fails
                    waitForQualityGate abortPipeline: true, credentialsId: 'sonar-token'
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished with status: ${currentBuild.currentResult}"
        }
        failure {
            echo "Pipeline failed! Stopping further execution."
        }
    }
}
