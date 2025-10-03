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
                    sh """
                        docker build -t ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} .
                        docker tag ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest
                    """
                }
            }
        }
        
        stage('Login to Docker Hub') {
            steps {
                script {
                    echo 'Logging into Docker Hub...'
                    sh """
                        echo \$DOCKERHUB_CREDENTIALS_PSW | docker login -u \$DOCKERHUB_CREDENTIALS_USR --password-stdin
                    """
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    echo "Pushing image to Docker Hub..."
                    sh """
                        docker push ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest
                    """
                }
            }
        }
        
        stage('Deploy to EC2') {
            steps {
                script {
                    echo "Deploying to EC2 instance: ${EC2_HOST}"
                    sh """
                        ssh -o StrictHostKeyChecking=no -i \$EC2_KEY ${EC2_USER}@${EC2_HOST} << 'EOF'
                            # Login to Docker Hub on EC2
                            echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin
                            
                            # Pull the latest image
                            docker pull ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}
                            
                            # Stop and remove old container if exists
                            docker stop ${CONTAINER_NAME} || true
                            docker rm ${CONTAINER_NAME} || true
                            
                            # Run new container
                            docker run -d --name ${CONTAINER_NAME} -p 80:80 ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}
                            
                            # Verify container is running
                            docker ps | grep ${CONTAINER_NAME}
EOF
                    """
                }
            }
        }
    }
    
    post {
        always {
            echo 'Cleaning up local Docker images...'
            sh """
                docker logout
                docker rmi ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} || true
                docker rmi ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest || true
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