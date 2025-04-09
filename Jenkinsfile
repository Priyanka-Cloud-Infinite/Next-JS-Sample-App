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
    environment {
        AWS_REGION = 'us-east-1' 
        CONTAINER_NAME = 'nextjs-app'
        APP_PORT = '3000'
    }
    steps {
        withCredentials([
            string(credentialsId: 'ecr-repository-url', variable: 'ECR_REPOSITORY_URL'), // e.g. 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app
            string(credentialsId: 'ec2-host-ip', variable: 'EC2_HOST_IP'),
            sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'SSH_KEY'),
            [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'ad8fe7e7-273b-40f0-a851-5fdaaa639e57', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
        ]) {
            sh '''
                # Extract ECR registry domain from the full repo URL
                ECR_REGISTRY=$(echo $ECR_REPOSITORY_URL | cut -d'/' -f1)

                # Create deployment script
                cat > deploy.sh << 'EOF'
#!/bin/bash
set -e

echo "Logging into ECR..."
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY

echo "Pulling Docker image..."
docker pull $ECR_REPOSITORY:latest

echo "Stopping and removing old container if it exists..."
docker stop $CONTAINER_NAME || true
docker rm $CONTAINER_NAME || true

echo "Running new container..."
docker run -d \\
  --name $CONTAINER_NAME \\
  --restart always \\
  -p $APP_PORT:3000 \\
  -e NODE_ENV=dev \\
  -v /opt/nextjs-app/logs:/app/logs \\
  $ECR_REPOSITORY:latest

echo "Deployment completed!"
EOF

                chmod +x deploy.sh

                echo "Copying script to EC2..."
                scp -i $SSH_KEY -o StrictHostKeyChecking=no deploy.sh ubuntu@$EC2_HOST_IP:/tmp/

                echo "Running deploy script remotely..."
                ssh -i $SSH_KEY -o StrictHostKeyChecking=no ubuntu@$EC2_HOST_IP bash -c "'
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    export AWS_REGION=$AWS_REGION
                    export ECR_REGISTRY=$ECR_REGISTRY
                    export ECR_REPOSITORY=$ECR_REPOSITORY_URL
                    export CONTAINER_NAME=$CONTAINER_NAME
                    export APP_PORT=$APP_PORT
                    sudo -E bash /tmp/deploy.sh
                '"

                echo "Cleaning up..."
                ssh -i $SSH_KEY -o StrictHostKeyChecking=no ubuntu@$EC2_HOST_IP 'rm -f /tmp/deploy.sh'
                rm -f deploy.sh
            '''
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
