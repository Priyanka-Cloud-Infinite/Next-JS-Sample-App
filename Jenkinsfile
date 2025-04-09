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
        withCredentials([
            string(credentialsId: 'ecr-repository-url', variable: 'ECR_REPOSITORY_URL'),
            string(credentialsId: 'ec2-host-ip', variable: 'EC2_HOST_IP'),
            sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'SSH_KEY'),
            [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
        ]) {
            sh '''
            # Create deployment script
            cat > deploy.sh << 'EOF'
#!/bin/bash
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
EOF

            # Make script executable
            chmod +x deploy.sh
            
            # Extract ECR registry
            ECR_REGISTRY=$(echo $ECR_REPOSITORY_URL | cut -d'/' -f1)
            
            # Copy script to EC2
            scp -i $SSH_KEY -o StrictHostKeyChecking=no deploy.sh ubuntu@$EC2_HOST_IP:/tmp/
            
            # Execute script on EC2
            ssh -i $SSH_KEY -o StrictHostKeyChecking=no ubuntu@$EC2_HOST_IP "
                export AWS_ACCESS_KEY_ID='$AWS_ACCESS_KEY_ID'
                export AWS_SECRET_ACCESS_KEY='$AWS_SECRET_ACCESS_KEY'
                export AWS_DEFAULT_REGION='$AWS_REGION'
                export ECR_REGISTRY='$ECR_REGISTRY'
                export ECR_REPOSITORY='$ECR_REPOSITORY_URL'
                export CONTAINER_NAME='nextjs-app'
                export APP_PORT='3000'
                sudo -E bash /tmp/deploy.sh
            "
            
            # Clean up
            ssh -i $SSH_KEY -o StrictHostKeyChecking=no ubuntu@$EC2_HOST_IP 'rm /tmp/deploy.sh'
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
