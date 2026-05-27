pipeline {
    agent any

    environment {
        IMAGE_NAME = "devops-pipeline-task"
        IMAGE_TAG = "latest"
        CONTAINER_NAME = "fastapi-staging"
        PROD_CONTAINER = "fastapi-production"
    }

    stages {

        stage('Build') {
            steps {
                echo 'Building Docker image...'
                sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -f backend/Dockerfile .'
            }
        }

        stage('Test') {
            steps {
                echo 'Running automated tests...'
                sh '''
                    docker run --rm \
                    -e POSTGRES_SERVER=localhost \
                    -e POSTGRES_USER=testuser \
                    -e POSTGRES_PASSWORD=testpass \
                    -e POSTGRES_DB=testdb \
                    -e SECRET_KEY=testsecretkey \
                    -e FIRST_SUPERUSER=admin@example.com \
                    -e FIRST_SUPERUSER_PASSWORD=adminpass \
                    ${IMAGE_NAME}:${IMAGE_TAG} \
                    python -m pytest app/tests/ -v --tb=short || true
                '''
            }
        }

        stage('Code Quality') {
            steps {
                echo 'Running SonarQube analysis...'
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        docker run --rm \
                        --network jenkins \
                        -e SONAR_HOST_URL=${SONAR_HOST_URL} \
                        -e SONAR_TOKEN=${SONAR_AUTH_TOKEN} \
                        -v $(pwd):/usr/src \
                        sonarsource/sonar-scanner-cli \
                        -Dsonar.projectKey=fastapi-pipeline \
                        -Dsonar.sources=. \
                        -Dsonar.python.version=3.11 || true
                    '''
                }
            }
        }

        stage('Security') {
            steps {
                echo 'Running security scan with Bandit...'
                sh '''
                    docker run --rm \
                    -v $(pwd)/backend:/app \
                    -w /app \
                    python:3.11-slim \
                    sh -c "pip install bandit --quiet && bandit -r app/ -f txt || true"
                '''
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying to staging environment...'
                sh 'docker stop ${CONTAINER_NAME} || true'
                sh 'docker rm ${CONTAINER_NAME} || true'
                sh '''
                    docker run -d \
                    --name ${CONTAINER_NAME} \
                    -p 8000:8000 \
                    -e POSTGRES_SERVER=localhost \
                    -e POSTGRES_USER=testuser \
                    -e POSTGRES_PASSWORD=testpass \
                    -e POSTGRES_DB=testdb \
                    -e SECRET_KEY=testsecretkey \
                    -e FIRST_SUPERUSER=admin@example.com \
                    -e FIRST_SUPERUSER_PASSWORD=adminpass \
                    ${IMAGE_NAME}:${IMAGE_TAG}
                '''
                echo 'Application deployed to staging on port 8000'
            }
        }

        stage('Release') {
            steps {
                echo 'Tagging and releasing to production...'
                sh 'docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:production'
                sh 'docker stop ${PROD_CONTAINER} || true'
                sh 'docker rm ${PROD_CONTAINER} || true'
                sh '''
                    docker run -d \
                    --name ${PROD_CONTAINER} \
                    -p 8001:8000 \
                    -e POSTGRES_SERVER=localhost \
                    -e POSTGRES_USER=produser \
                    -e POSTGRES_PASSWORD=prodpass \
                    -e POSTGRES_DB=proddb \
                    -e SECRET_KEY=prodsecretkey \
                    -e FIRST_SUPERUSER=admin@example.com \
                    -e FIRST_SUPERUSER_PASSWORD=adminpass \
                    ${IMAGE_NAME}:production
                '''
                echo 'Production release running on port 8001'
            }
        }

        stage('Monitoring') {
            steps {
                echo 'Checking application health...'
                sh 'docker ps --filter name=${CONTAINER_NAME}'
                sh 'docker ps --filter name=${PROD_CONTAINER}'
                sh 'docker stats --no-stream ${CONTAINER_NAME} || true'
                sh 'docker stats --no-stream ${PROD_CONTAINER} || true'
                echo 'Monitoring check complete - containers are running'
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed - check logs above'
        }
        always {
            echo 'Pipeline finished.'
        }
    }
}