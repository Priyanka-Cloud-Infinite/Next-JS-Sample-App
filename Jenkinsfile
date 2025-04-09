pipeline {
    agent { label 'build-agent' }

    environment {
        ECR_REPOSITORY = credentials('ecr-repository-url')
        AWS_REGION = 'us-east-1'
        SONAR_HOST_URL = 'http://13.219.61.104:9000'
        SONAR_TOKEN = credentials('sonarqube-token') 
        APP_NAME = 'nextjs-app'
        APP_VERSION = "${env.BUILD_NUMBER}"
        DOCKER_BUILDKIT = '1'  // Enable BuildKit
    }

    stages {
        stage('Deploy to EC2') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'SSH_KEY')]) {
                    script {
                        // Create a deployment script without interpolating secrets
                        writeFile file: 'deploy.sh', text: '''#!/bin/bash
set -e

# Login to ECR
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY

# Pull the latest image
docker pull $ECR_REPOSITORY:latest

# Stop and remove existing container
docker stop $CONTAINER_NAME || true
docker rm $CONTAINER_NAME || true

# Run the new container
docker run -d \\
  --name $CONTAINER_NAME \\
  --restart always \\
  -p $APP_PORT:3000 \\
  -e NODE_ENV=dev \\
  -v /opt/nextjs-app/logs:/app/logs \\
  $ECR_REPOSITORY:latest

echo "Deployment completed successfully!"
'''

                        // Make the script executable
                        sh 'chmod +x deploy.sh'
                        
                        // Copy the script to the EC2 instance
                        sh "scp -i ${SSH_KEY} -o StrictHostKeyChecking=no deploy.sh ${SSH_USER}@${EC2_HOST}:/tmp/"
                        
                        // Extract ECR registry for secure passing
                        def ecrRegistry = sh(script: "echo ${ECR_REPOSITORY} | cut -d'/' -f1", returnStdout: true).trim()
                        
                        // Execute the script on the EC2 instance with AWS credentials
                        withAWS(credentials: 'aws-key', region: "${AWS_REGION}") {
                            sh """
                            ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${EC2_HOST} '
                            export AWS_ACCESS_KEY_ID=\$(aws configure get aws_access_key_id)
                            export AWS_SECRET_ACCESS_KEY=\$(aws configure get aws_secret_access_key)
                            export AWS_DEFAULT_REGION=${AWS_REGION}
                            export ECR_REGISTRY=${ecrRegistry}
                            export ECR_REPOSITORY=${ECR_REPOSITORY}
                            export CONTAINER_NAME=${CONTAINER_NAME}
                            export APP_PORT=${APP_PORT}
                            sudo -E bash /tmp/deploy.sh
                            '
                            """
                        }
                        
                        // Clean up the script on the EC2 instance
                        sh "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${EC2_HOST} 'rm /tmp/deploy.sh'"
                    }
                }
            }
        }
    }

    post {
        always {
            // Clean up Docker images
            sh """
            docker rmi ${APP_NAME}:${APP_VERSION} || true
            docker rmi ${ECR_REPOSITORY}:${APP_VERSION} || true
            docker rmi ${ECR_REPOSITORY}:latest || true
            """
            
            // Clean up deployment script
            sh 'rm -f deploy.sh'
        }
        
        success {
            echo "Deployment completed successfully! The application is now running on ${EC2_HOST}:${APP_PORT}"
        }
    }
}
