pipeline {
    agent any

    tools {
        maven 'maven3'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        MAVEN_DIR = "database_service_project"
        DOCKER_IMAGE = "fazil2664/boardshack"
        DOCKER_TAG = "latest"
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                credentialsId: 'git-cred',
                url: 'https://github.com/fasil7170/Boardgame.git'
            }
        }

        stage('Verify Environment') {
            steps {
                sh '''
                java -version
                mvn -version
                docker --version
                '''
            }
        }

        stage('Validate POM') {
            steps {
                dir("${MAVEN_DIR}") {
                    sh "mvn validate"
                }
            }
        }

        stage('Build Application') {
            steps {
                dir("${MAVEN_DIR}") {
                    sh "mvn clean package -DskipTests"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                dir("${MAVEN_DIR}") {
                    withSonarQubeEnv('sonar') {
                        sh """
                        mvn sonar:sonar \
                        -Dsonar.projectKey=BoardGame \
                        -Dsonar.projectName=BoardGame
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Publish To Nexus') {
            steps {
                dir("${MAVEN_DIR}") {
                    withMaven(globalMavenSettingsConfig: 'global-settings', maven: 'maven3') {
                        sh "mvn deploy -DskipTests"
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                dir("${MAVEN_DIR}") {
                    sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                }
            }
        }

        stage('Docker Push') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred', url: '') {
                    sh "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'k8-cred') {
                    sh """
                    kubectl apply -f ${MAVEN_DIR}/deployment-service.yaml
                    kubectl rollout status deployment/boardgame -n webapps
                    """
                }
            }
        }

    }

    post {

        success {
            echo "Pipeline completed successfully 🎉"
        }

        failure {
            echo "Pipeline failed ❌"
        }

        always {
            archiveArtifacts artifacts: "${MAVEN_DIR}/target/*.jar"
        }
    }
}
