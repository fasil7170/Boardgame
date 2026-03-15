pipeline {
    agent any

    tools {
        jdk 'jdk21'      // Ensure JDK 21 installed in Jenkins Global Tool Config
        maven 'maven3'   // Ensure Maven 3.9+ installed in Jenkins Global Tool Config
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        PATH = "${tool 'jdk21'}/bin:${tool 'maven3'}/bin:${env.PATH}"
    }

    options {
        skipDefaultCheckout(true)
        timeout(time: 60, unit: 'MINUTES') // Stop long-running builds
    }

    stages {

        stage('Verify Tools') {
            steps {
                sh '''
                    echo "JAVA_HOME=${JAVA_HOME}"
                    java -version
                    mvn -version
                    $SCANNER_HOME/bin/sonar-scanner -v || true
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
                sh 'mvn clean compile -Djava.version=21'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test -Djava.version=21'
            }
        }

        stage('File System Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
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
                sh 'mvn package -Djava.version=21'
            }
        }

        stage('Publish To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', maven: 'maven3') {
                    sh 'mvn deploy -Djava.version=21'
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                    sh 'docker build -t fazil2664/boardshack:latest .'
                    sh 'docker push fazil2664/boardshack:latest'
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'k8-cred', namespace: 'webapps') {
                    sh 'kubectl apply -f deployment-service.yaml'
                    sh 'kubectl get pods -n webapps'
                    sh 'kubectl get svc -n webapps'
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished with status: ${currentBuild.currentResult}"
        }
        failure {
            error "Pipeline failed. Stopping further stages."
        }
    }
}
