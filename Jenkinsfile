pipeline {


agent any

options {
    timeout(time: 60, unit: 'MINUTES')
    disableConcurrentBuilds()
}

tools {
    jdk 'jdk17'
    maven 'maven3'
}

environment {
    IMAGE_NAME = "fazil2664/boardshack"
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

    stage('Compile') {
        steps {
            sh 'mvn clean compile'
        }
    }

    stage('Unit Test') {
        steps {
            sh 'mvn test'
        }
    }

    stage('SonarQube Analysis') {
        steps {
            withSonarQubeEnv('sonar') {
                sh """
                ${SCANNER_HOME}/bin/sonar-scanner \
                -Dsonar.projectKey=BoardGame \
                -Dsonar.projectName=BoardGame \
                -Dsonar.java.binaries=target/classes
                """
            }
        }
    }

    stage('Quality Gate') {
        steps {
            script {
                waitForQualityGate abortPipeline: true
            }
        }
    }

    stage('Build Artifact') {
        steps {
            sh 'mvn package -DskipTests'
        }
    }

    stage('Publish to Nexus') {
        steps {
            withMaven(
                maven: 'maven3',
                jdk: 'jdk17',
                globalMavenSettingsConfig: 'global-settings'
            ) {
                sh 'mvn deploy -DskipTests'
            }
        }
    }

    stage('Build Docker Image') {
        steps {
            sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} ."
            sh "docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest"
        }
    }

    stage('Trivy Security Scan') {
        steps {
            sh "trivy image ${IMAGE_NAME}:${BUILD_NUMBER}"
        }
    }

    stage('Push Docker Image') {
        steps {
            script {
                withDockerRegistry(credentialsId: 'docker-cred') {

                    sh "docker push ${IMAGE_NAME}:${BUILD_NUMBER}"
                    sh "docker push ${IMAGE_NAME}:latest"

                }
            }
        }
    }

    stage('Deploy to Kubernetes') {
        steps {

            withKubeConfig(
                credentialsId: 'k8-cred',
                namespace: 'webapps',
                serverUrl: 'https://192.168.0.100:6443'
            ) {

                sh 'kubectl apply -f deployment-service.yaml'
                sh 'kubectl rollout status deployment/boardgame-deployment -n webapps'

            }

        }
    }

    stage('Verify Deployment') {
        steps {

            withKubeConfig(
                credentialsId: 'k8-cred',
                namespace: 'webapps',
                serverUrl: 'https://192.168.0.100:6443'
            ) {

                sh 'kubectl get pods -n webapps'
                sh 'kubectl get svc -n webapps'

            }

        }
    }

}

post {

    success {
        mail(
            to: 'rkf@gmail.com',
            subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: "Build completed successfully.\n${env.BUILD_URL}"
        )
    }

    failure {
        mail(
            to: 'rkf@gmail.com',
            subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: "Build failed.\n${env.BUILD_URL}"
        )
    }

}
```

}
