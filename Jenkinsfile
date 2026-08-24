pipeline {
    agent any

    environment {
        IMAGE_NAME = "localhost:5000/demo-app"
        IMAGE_TAG  = "${env.BUILD_NUMBER}"
        APP_PORT   = "8083"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn -B clean verify'
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Docker Push') {
            steps {
                sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Deploy (verify)') {
            steps {
                sh """
                    docker rm -f demo-app || true
                    docker run -d --name demo-app -p ${APP_PORT}:8080 ${IMAGE_NAME}:${IMAGE_TAG}
                    sleep 5
                    curl -sf localhost:${APP_PORT}/hello
                """
            }
        }
    }
}
