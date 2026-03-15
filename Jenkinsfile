pipeline {
    agent any

    tools {
        maven 'maven3'   // Ensure Maven 3.9+ installed in Jenkins
        jdk 'jdk21'      // Ensure JDK 21 installed in Jenkins
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        JAVA_HOME = "${tool 'jdk21'}"
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
    }

    options {
        skipDefaultCheckout(true)
        timeout(time: 1, unit: 'HOURS') // optional: prevent hanging builds
    }

    stages {
        stage('Verify Tools') {
            steps {
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
                dir("${env.WORKSPACE}") {
                    sh "mvn clean compile -Djava.version=21"
                }
            }
        }

        stage('Test') {
            steps {
                dir("${env.WORKSPACE}") {
                    sh "mvn test -Djava.version=21"
                }
            }
        }

        stage('Package') {
            steps {
                dir("${env.WORKSPACE}") {
                    sh "mvn package -Djava.version=21"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                dir("${env.WORKSPACE}") {
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
                    waitForQualityGate abortPipeline: true, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Publish To Nexus') {
            steps {
                dir("${env.WORKSPACE}") {
                    withMaven(globalMavenSettingsConfig: 'global-settings', maven: 'maven3') {
                        sh "mvn deploy -Djava.version=21"
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                dir("${env.WORKSPACE}") {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t fazil2664/boardshack:latest ."
                        sh "docker push fazil2664/boardshack:latest"
                    }
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                dir("${env.WORKSPACE}") {
                    withKubeConfig(credentialsId: 'k8-cred', namespace: 'webapps') {
                        sh "kubectl apply -f deployment-service.yaml"
                        sh "kubectl get pods -n webapps"
                        sh "kubectl get svc -n webapps"
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished with status: ${currentBuild.currentResult}"
        }
        failure {
            error("Pipeline failed. Stopping further execution.")
        }
    }
}
