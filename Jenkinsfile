pipeline {
    agent any

    tools {
        maven 'maven3' // Ensure Maven is installed in Jenkins
    }

    options {
        skipDefaultCheckout(true) // We'll do Git checkout manually
        timeout(time: 30, unit: 'MINUTES') // Optional: fail if pipeline hangs
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner' // Sonar scanner installed in Jenkins
        MAVEN_DIR = 'database_service_project' // Change this to where your pom.xml is
    }

    stages {
        stage('Verify Tools') {
            steps {
                script {
                    // Detect JAVA_HOME dynamically
                    env.JAVA_HOME = sh(script: "readlink -f \$(which javac) | sed 's:/bin/javac::'", returnStdout: true).trim()
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

        stage('Validate POM') {
            steps {
                script {
                    if (!fileExists("${env.MAVEN_DIR}/pom.xml")) {
                        error "pom.xml not found in ${env.MAVEN_DIR}, cannot proceed!"
                    }
                }
            }
        }

        stage('Compile') {
            steps {
                dir("${env.MAVEN_DIR}") {
                    sh "mvn compile"
                }
            }
        }

        stage('Test') {
            steps {
                dir("${env.MAVEN_DIR}") {
                    sh "mvn test"
                }
            }
        }

        stage('Package') {
            steps {
                dir("${env.MAVEN_DIR}") {
                    sh "mvn package"
                }
            }
        }

        stage('Publish To Nexus') {
            steps {
                dir("${env.MAVEN_DIR}") {
                    withMaven(globalMavenSettingsConfig: 'global-settings', maven: 'maven3') {
                        sh "mvn deploy"
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
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
                dir("${env.MAVEN_DIR}") {
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
                waitForQualityGate abortPipeline: true, credentialsId: 'sonar-token'
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
