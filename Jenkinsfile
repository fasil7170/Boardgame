pipeline {

```
agent any

options {
    timestamps()
    timeout(time: 60, unit: 'MINUTES')
}

tools {
    jdk 'jdk17'
    maven 'maven3'
}

environment {
    SCANNER_HOME = tool 'sonar-scanner'
    MAVEN_OPTS = "-Xmx1024m -XX:+UseG1GC"
    IMAGE_NAME = "fazil2664/boardshack"
}

stages {

    // =========================
    // 1. Checkout Source Code
    // =========================
    stage('Checkout') {
        steps {
            git branch: 'main',
                credentialsId: 'git-cred',
                url: 'https://github.com/fasil7170/Boardgame.git'
        }
    }

    // =========================
    // 2. Compile Application
    // =========================
    stage('Compile') {
        steps {
            sh '''
            mvn clean compile \
            -Dmaven.compiler.source=17 \
            -Dmaven.compiler.target=17
            '''
        }
    }

    // =========================
    // 3. Run Unit Tests
    // =========================
    stage('Unit Tests') {
        steps {
            sh '''
            mvn test \
            -B \
            -T 2C \
            -Dspring.main.lazy-initialization=true \
            -Dspring.main.web-application-type=none \
            -Dspring.jpa.open-in-view=false
            '''
        }
    }

    // =========================
    // 4. Security Scanning
    // =========================
    stage('Security Scans') {
        parallel {

            stage('Filesystem Scan') {
                steps {
                    sh 'trivy fs --format table -o trivy-fs-report.html .'
                }
            }

            stage('Docker Image Scan') {
                steps {
                    script {
                        sh "docker build -t ${IMAGE_NAME}:latest ."
                        sh "trivy image --format table -o trivy-image-report.html ${IMAGE_NAME}:latest"
                    }
                }
            }

        }
    }

    // =========================
    // 5. SonarQube Analysis
    // =========================
    stage('SonarQube Analysis') {
        steps {
            withSonarQubeEnv('sonar') {
                sh """
                ${SCANNER_HOME}/bin/sonar-scanner \
                -Dsonar.projectName=BoardGame \
                -Dsonar.projectKey=BoardGame \
                -Dsonar.java.binaries=target/classes
                """
            }
        }
    }

    // =========================
    // 6. Quality Gate
    // =========================
    stage('Quality Gate') {
        steps {
            script {
                def qg = waitForQualityGate()

                if (qg.status != 'OK') {
                    error "Quality Gate failed: ${qg.status}"
                }
            }
        }
    }

    // =========================
    // 7. Package Application
    // =========================
    stage('Package') {
        steps {
            sh 'mvn package -DskipTests'
        }
    }

    // =========================
    // 8. Publish Artifact to Nexus
    // =========================
    stage('Publish to Nexus') {
        steps {
            withMaven(
                globalMavenSettingsConfig: 'global-settings',
                jdk: 'jdk17',
                maven: 'maven3',
                traceability: true
            ) {
                sh 'mvn deploy -DskipTests'
            }
        }
    }

    // =========================
    // 9. Push Docker Image
    // =========================
    stage('Push Docker Image') {
        steps {
            script {

                sh "docker tag ${IMAGE_NAME}:latest ${IMAGE_NAME}:${BUILD_NUMBER}"

                withDockerRegistry(credentialsId: 'docker-cred') {

                    sh "docker push ${IMAGE_NAME}:latest"
                    sh "docker push ${IMAGE_NAME}:${BUILD_NUMBER}"

                }
            }
        }
    }

    // =========================
    // 10. Deploy to Kubernetes
    // =========================
    stage('Deploy to Kubernetes') {
        steps {
            script {

                withKubeConfig(
                    credentialsId: 'k8-cred',
                    namespace: 'webapps',
                    serverUrl: 'https://192.168.0.100:6443'
                ) {

                    try {

                        sh 'kubectl apply -f deployment-service.yaml'
                        sh 'kubectl rollout status deployment/boardgame-deployment -n webapps'

                    } catch (Exception e) {

                        echo "Deployment failed. Rolling back..."

                        sh 'kubectl rollout undo deployment/boardgame-deployment -n webapps'

                        error "Rollback executed due to deployment failure"

                    }

                }
            }
        }
    }

    // =========================
    // 11. Verify Deployment
    // =========================
    stage('Verify Deployment') {
        steps {

            withKubeConfig(
                credentialsId: 'k8-cred',
                namespace: 'webapps',
                serverUrl: 'https://192.168.0.100:6443'
            ) {
```
