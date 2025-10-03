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
        
        stage('Transfer Files to EC2') {
            steps {
                script {
                    echo "Transferring files to EC2..."
                    withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'SSH_KEY')]) {
                        bat """
                            echo "Testing SSH connection..."
                            ssh -i "%SSH_KEY%" -o StrictHostKeyChecking=accept-new %EC2_USER%@%EC2_HOST% "echo 'SSH connection successful'"
                            
                            echo "Copying files..."
                            scp -i "%SSH_KEY%" -o StrictHostKeyChecking=accept-new Dockerfile index.html %EC2_USER%@%EC2_HOST%:/home/%EC2_USER%/
                        """
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image on EC2: ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}"
                    sshagent(credentials: ['ec2-ssh-key']) {
                        bat """
                            ssh -o StrictHostKeyChecking=no %EC2_USER%@%EC2_HOST% "cd /home/%EC2_USER% && docker build -t %DOCKERHUB_USERNAME%/%IMAGE_NAME%:%IMAGE_TAG% . && docker tag %DOCKERHUB_USERNAME%/%IMAGE_NAME%:%IMAGE_TAG% %DOCKERHUB_USERNAME%/%IMAGE_NAME%:latest"
                        """
                    }
                }
            }
        }
        
        stage('Login to Docker Hub') {
            steps {
                script {
                    echo 'Logging into Docker Hub on EC2...'
                    sshagent(credentials: ['ec2-ssh-key']) {
                        withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
                            bat """
                                ssh -o StrictHostKeyChecking=no %EC2_USER%@%EC2_HOST% "echo %DOCKERHUB_PASS% | docker login -u %DOCKERHUB_USER% --password-stdin"
                            """
                        }
                    }
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    echo "Pushing image to Docker Hub from EC2..."
                    sshagent(credentials: ['ec2-ssh-key']) {
                        bat """
                            ssh -o StrictHostKeyChecking=no %EC2_USER%@%EC2_HOST% "docker push %DOCKERHUB_USERNAME%/%IMAGE_NAME%:%IMAGE_TAG% && docker push %DOCKERHUB_USERNAME%/%IMAGE_NAME%:latest"
                        """
                    }
                }
            }
        }
        
        stage('Deploy Container') {
            steps {
                script {
                    echo "Deploying container on EC2: ${EC2_HOST}"
                    sshagent(credentials: ['ec2-ssh-key']) {
                        bat """
                            ssh -o StrictHostKeyChecking=no %EC2_USER%@%EC2_HOST% "
                                docker stop %CONTAINER_NAME% || true &&
                                docker rm %CONTAINER_NAME% || true &&
                                docker run -d --name %CONTAINER_NAME% -p 80:80 %DOCKERHUB_USERNAME%/%IMAGE_NAME%:%IMAGE_TAG% &&
                                docker ps | grep %CONTAINER_NAME%
                            "
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo 'Cleaning up Docker images on EC2...'
            withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'SSH_KEY')]) {
                bat """
                    ssh -i "%SSH_KEY%" -o StrictHostKeyChecking=accept-new %EC2_USER%@%EC2_HOST% "
                        docker logout || true
                        docker rmi %DOCKERHUB_USERNAME%/%IMAGE_NAME%:%IMAGE_TAG% || true
                        docker rmi %DOCKERHUB_USERNAME%/%IMAGE_NAME%:latest || true
                    "
                """
            }
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