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
                    docker rm -f demo-app demo-db-ci || true
                    docker network create demo-ci-net || true

                    docker run -d --name demo-db-ci --network demo-ci-net \\
                        -e POSTGRES_DB=demo -e POSTGRES_USER=demo -e POSTGRES_PASSWORD=demo \\
                        -v \$(pwd)/init.sql:/docker-entrypoint-initdb.d/init.sql:ro \\
                        postgres:16
                    until docker exec demo-db-ci pg_isready -U demo -d demo; do sleep 2; done

                    docker run -d --name demo-app --network demo-ci-net -p ${APP_PORT}:8080 \\
                        -e DB_HOST=demo-db-ci -e DB_NAME=demo -e DB_USER=demo -e DB_PASSWORD=demo \\
                        ${IMAGE_NAME}:${IMAGE_TAG}
                    APP_IP=\$(docker inspect -f '{{.NetworkSettings.Networks.demo-ci-net.IPAddress}}' demo-app)
                    curl -sf --retry 10 --retry-delay 2 --retry-connrefused http://\${APP_IP}:8080/hello
                """
            }
        }
    }
}
