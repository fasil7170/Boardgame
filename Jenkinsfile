pipeline {
agent any

```
tools {
    maven 'maven3'
}

environment {
    SCANNER_HOME = tool 'sonar-scanner'
}

stages {

    stage('Git Checkout') {
        steps {
            git branch: 'main',
                credentialsId: 'git-cred',
                url: 'https://github.com/fasil7170/Boardgame.git'
        }
    }

    stage('Verify Tools') {
        steps {
            sh '''
            java -version
            mvn -version
            '''
        }
    }

    stage('Compile') {
        steps {
            sh "mvn compile"
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
            waitForQualityGate abortPipeline: false
        }
    }

    stage('Build') {
        steps {
            sh "mvn package"
        }
    }

    stage('Publish To Nexus') {
        steps {
            withMaven(globalMavenSettingsConfig: 'global-settings') {
                sh "mvn deploy"
            }
        }
    }

    stage('Build Docker Image') {
        steps {
            sh "docker build -t fazil2664/boardshack:latest ."
        }
    }

    stage('Docker Image Scan') {
        steps {
            sh "trivy image --format table -o trivy-image-report.html fazil2664/boardshack:latest"
        }
    }

    stage('Push Docker Image') {
        steps {
            withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                sh "docker push fazil2664/boardshack:latest"
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
        echo "Pipeline completed with status: ${currentBuild.currentResult}"
    }
}
```

}
