pipeline {
    agent any
    
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKERHUB_USERNAME = 'mohamedtarek123'
        IMAGE_NAME = 'nginx-custom-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        EC2_HOST = '56.228.11.117'
        EC2_USER = 'mohamedtarek112'
        EC2_KEY = credentials('ec2-ssh-key')
        CONTAINER_NAME = 'nginx-app'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image: ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}"
                    bat """
                        docker build -t %DOCKERHUB_USERNAME%/%IMAGE_NAME%:%IMAGE_TAG% .
                        docker tag %DOCKERHUB_USERNAME%/%IMAGE_NAME%:%IMAGE_TAG% %DOCKERHUB_USERNAME%/%IMAGE_NAME%:latest
                    """
                }
            }
        }
        
        stage('Login to Docker Hub') {
            steps {
                script {
                    echo 'Logging into Docker Hub...'
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
                        bat "echo %DOCKERHUB_PASS% | docker login -u %DOCKERHUB_USER% --password-stdin"
                    }
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    echo "Pushing image to Docker Hub..."
                    bat """
                        docker push %DOCKERHUB_USERNAME%/%IMAGE_NAME%:%IMAGE_TAG%
                        docker push %DOCKERHUB_USERNAME%/%IMAGE_NAME%:latest
                    """
                }
            }
        }
        
        stage('Deploy to EC2') {
            steps {
                script {
                    echo "Deploying to EC2 instance: ${EC2_HOST}"
                    withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'KEY_FILE')]) {
                        bat """
                            ssh -o StrictHostKeyChecking=no -i %KEY_FILE% %EC2_USER%@%EC2_HOST% "
                            docker login -u %DOCKERHUB_USERNAME% -p %DOCKERHUB_CREDENTIALS_PSW% &&
                            docker pull %DOCKERHUB_USERNAME%/%IMAGE_NAME%:%IMAGE_TAG% &&
                            docker stop %CONTAINER_NAME% || true &&
                            docker rm %CONTAINER_NAME% || true &&
                            docker run -d --name %CONTAINER_NAME% -p 80:80 %DOCKERHUB_USERNAME%/%IMAGE_NAME%:%IMAGE_TAG% &&
                            docker ps | findstr %CONTAINER_NAME%
                            "
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo 'Cleaning up local Docker images...'
            bat """
                docker logout
                docker rmi %DOCKERHUB_USERNAME%/%IMAGE_NAME%:%IMAGE_TAG% || true
                docker rmi %DOCKERHUB_USERNAME%/%IMAGE_NAME%:latest || true
            """
        }
        success {
            echo 'Pipeline completed successfully!'
            echo "Application is accessible at: http://${EC2_HOST}"
        }
        failure {
            echo 'Pipeline failed. Please check the logs.'
        }
    }
}