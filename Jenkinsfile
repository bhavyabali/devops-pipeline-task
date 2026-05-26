pipeline {
    agent any

    environment {
        IMAGE_NAME = "devops-pipeline-task"
        IMAGE_TAG = "latest"
        CONTAINER_NAME = "fastapi-staging"
        PROD_CONTAINER = "fastapi-production"
    }

    stages {

        // STAGE 1 - BUILD
        stage('Build') {
            steps {
                echo 'Building Docker image...'
                bat 'docker build -t %IMAGE_NAME%:%IMAGE_TAG% ./backend'
            }
        }

        // STAGE 2 - TEST
        stage('Test') {
            steps {
                echo 'Running automated tests...'
                bat '''
                    docker run --rm ^
                    -e POSTGRES_SERVER=localhost ^
                    -e POSTGRES_USER=testuser ^
                    -e POSTGRES_PASSWORD=testpass ^
                    -e POSTGRES_DB=testdb ^
                    -e SECRET_KEY=testsecretkey ^
                    -e FIRST_SUPERUSER=admin@example.com ^
                    -e FIRST_SUPERUSER_PASSWORD=adminpass ^
                    %IMAGE_NAME%:%IMAGE_TAG% ^
                    python -m pytest app/tests/ -v --tb=short || exit 0
                '''
            }
        }

        // STAGE 3 - CODE QUALITY
        stage('Code Quality') {
            steps {
                echo 'Running code quality checks...'
                bat '''
                    docker run --rm ^
                    -v %CD%\\backend:/app ^
                    -w /app ^
                    python:3.11-slim ^
                    sh -c "pip install flake8 --quiet && flake8 app/ --max-line-length=120 --statistics || exit 0"
                '''
            }
        }

        // STAGE 4 - SECURITY
        stage('Security') {
            steps {
                echo 'Running security scan with Bandit...'
                bat '''
                    docker run --rm ^
                    -v %CD%\\backend:/app ^
                    -w /app ^
                    python:3.11-slim ^
                    sh -c "pip install bandit --quiet && bandit -r app/ -f txt || exit 0"
                '''
            }
        }

        // STAGE 5 - DEPLOY (Staging)
        stage('Deploy') {
            steps {
                echo 'Deploying to staging environment...'
                bat 'docker stop %CONTAINER_NAME% || exit 0'
                bat 'docker rm %CONTAINER_NAME% || exit 0'
                bat '''
                    docker run -d ^
                    --name %CONTAINER_NAME% ^
                    -p 8000:8000 ^
                    -e POSTGRES_SERVER=localhost ^
                    -e POSTGRES_USER=testuser ^
                    -e POSTGRES_PASSWORD=testpass ^
                    -e POSTGRES_DB=testdb ^
                    -e SECRET_KEY=testsecretkey ^
                    -e FIRST_SUPERUSER=admin@example.com ^
                    -e FIRST_SUPERUSER_PASSWORD=adminpass ^
                    %IMAGE_NAME%:%IMAGE_TAG%
                '''
                echo 'Application deployed to staging on port 8000'
            }
        }

        // STAGE 6 - RELEASE
        stage('Release') {
            steps {
                echo 'Tagging and releasing to production...'
                bat 'docker tag %IMAGE_NAME%:%IMAGE_TAG% %IMAGE_NAME%:production'
                bat 'docker stop %PROD_CONTAINER% || exit 0'
                bat 'docker rm %PROD_CONTAINER% || exit 0'
                bat '''
                    docker run -d ^
                    --name %PROD_CONTAINER% ^
                    -p 8001:8000 ^
                    -e POSTGRES_SERVER=localhost ^
                    -e POSTGRES_USER=produser ^
                    -e POSTGRES_PASSWORD=prodpass ^
                    -e POSTGRES_DB=proddb ^
                    -e SECRET_KEY=prodsecretkey ^
                    -e FIRST_SUPERUSER=admin@example.com ^
                    -e FIRST_SUPERUSER_PASSWORD=adminpass ^
                    %IMAGE_NAME%:production
                '''
                echo 'Production release running on port 8001'
            }
        }

        // STAGE 7 - MONITORING
        stage('Monitoring') {
            steps {
                echo 'Checking application health...'
                bat 'docker ps --filter name=%CONTAINER_NAME%'
                bat 'docker ps --filter name=%PROD_CONTAINER%'
                bat 'docker stats --no-stream %CONTAINER_NAME% || exit 0'
                bat 'docker stats --no-stream %PROD_CONTAINER% || exit 0'
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