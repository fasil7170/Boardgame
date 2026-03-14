pipeline {

agent {
    kubernetes {
        yaml """
apiVersion: v1
kind: Pod
spec:
  containers:

  - name: maven
    image: maven:3.9.9-eclipse-temurin-17
    command:
    - cat
    tty: true

  - name: docker
    image: docker:25
    command:
    - cat
    tty: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock

  - name: trivy
    image: aquasec/trivy
    command:
    - cat
    tty: true

  - name: kubectl
    image: bitnami/kubectl
    command:
    - cat
    tty: true

  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
"""
    }
}

environment {
    IMAGE_NAME = "fazil2664/boardshack"
}

stages {

stage('Checkout Code') {
    steps {
        git branch: 'main',
        credentialsId: 'git-cred',
        url: 'https://github.com/fasil7170/Boardgame.git'
    }
}

stage('Build & Test') {
    steps {
        container('maven') {
            sh 'mvn clean package'
        }
    }
}

stage('Trivy File Scan') {
    steps {
        container('trivy') {
            sh 'trivy fs .'
        }
    }
}

stage('SonarQube Analysis') {
    steps {
        container('maven') {
            withSonarQubeEnv('sonar') {
                sh '''
                mvn sonar:sonar \
                -Dsonar.projectKey=BoardGame \
                -Dsonar.projectName=BoardGame
                '''
            }
        }
    }
}

stage('Quality Gate') {
    steps {
        waitForQualityGate abortPipeline: false
    }
}

stage('Build Docker Image') {
    steps {
        container('docker') {
            sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} ."
        }
    }
}

stage('Scan Docker Image') {
    steps {
        container('trivy') {
            sh "trivy image ${IMAGE_NAME}:${BUILD_NUMBER}"
        }
    }
}

stage('Push Docker Image') {
    steps {
        container('docker') {
            withDockerRegistry(url: 'https://index.docker.io/v1/', credentialsId: 'docker-cred') {
                sh "docker push ${IMAGE_NAME}:${BUILD_NUMBER}"
            }
        }
    }
}

stage('Deploy to Kubernetes') {
    steps {
        container('kubectl') {
            withKubeConfig(credentialsId: 'k8-cred', namespace: 'webapps') {
                sh 'kubectl apply -f deployment-service.yaml'
            }
        }
    }
}

stage('Verify Deployment') {
    steps {
        container('kubectl') {
            sh 'kubectl get pods'
            sh 'kubectl get svc'
        }
    }
}

}

post {
    success {
        echo "Pipeline completed successfully"
    }
    failure {
        echo "Pipeline failed"
    }
}

}
