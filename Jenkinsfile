pipeline {
              agent any
              
              environment {
                  ECR_REPOSITORY = credentials('ecr-repository-url')
                  AWS_REGION = 'us-east-1'
                  SONAR_HOST_URL = 'http://sonarqube:9000'
                  SONAR_TOKEN = credentials('sonarqube-token') 
                  MONGODB_URI_DEV = credentials('mongodb-uri-dev')
                  APP_NAME = 'nextjs-app'
                  APP_VERSION = "${env.BUILD_NUMBER}"
              }
              
              stages {
                  stage('Checkout') {
                      steps {
                          checkout scm
                      }
                  }
                  
                  stage('Install Dependencies') {
                      steps {
                          sh 'npm ci'
                      }
                  }
                  
                  stage('Code Quality Analysis') {
                      steps {
                          withSonarQubeEnv('SonarQube') {
                              sh '''
                              sonar-scanner \\
                                -Dsonar.projectKey=${APP_NAME} \\
                                -Dsonar.projectName=${APP_NAME} \\
                                -Dsonar.sources=. \\
                                -Dsonar.exclusions=node_modules/**,**/*.test.js,**/*.spec.js \\
                                -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
                              '''
                          }
                          
                          timeout(time: 10, unit: 'MINUTES') {
                              waitForQualityGate abortPipeline: true
                          }
                      }
                  }
                  
                  stage('Run Tests') {
                      steps {
                          sh 'npm test'
                      }
                  }
                  
                  stage('Build Application') {
                      steps {
                          sh 'npm run build'
                      }
                  }
                  
                  stage('Build Docker Image') {
                      steps {
                          script {
                              // Create Dockerfile if it doesn't exist
                              sh '''
                              if [ ! -f Dockerfile ]; then
                                cat > Dockerfile << 'EOF'
          FROM node:18-alpine AS builder
          WORKDIR /app
          COPY package*.json ./
          RUN npm ci
          COPY . .
          RUN npm run build
          
          FROM node:18-alpine AS runner
          WORKDIR /app
          ENV NODE_ENV=production
          COPY --from=builder /app/package*.json ./
          COPY --from=builder /app/next.config.js ./
          COPY --from=builder /app/public ./public
          COPY --from=builder /app/.next ./.next
          COPY --from=builder /app/node_modules ./node_modules
          
          EXPOSE 3000
          CMD ["npm", "start"]
          EOF
                              fi
                              '''
                              
                              // Build Docker image
                              sh '''
                              docker build -t ${APP_NAME}:${APP_VERSION} .
                              '''
                          }
                      }
                  }
                  
                  stage('Scan Docker Image') {
                      steps {
                          script {
                              // Scan with Trivy
                              sh '''
                              trivy image --severity HIGH,CRITICAL --exit-code 1 ${APP_NAME}:${APP_VERSION}
                              '''
                          }
                      }
                  }
                  
                  stage('Push to ECR') {
                      steps {
                          script {
                              // Login to ECR
                              sh '''
                              aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPOSITORY}
                              '''
                              
                              // Tag and push image
                              sh '''
                              docker tag ${APP_NAME}:${APP_VERSION} ${ECR_REPOSITORY}:${APP_VERSION}
                              docker tag ${APP_NAME}:${APP_VERSION} ${ECR_REPOSITORY}:latest
                              docker push ${ECR_REPOSITORY}:${APP_VERSION}
                              docker push ${ECR_REPOSITORY}:latest
                              '''
                          }
                      }
                  }
                  
                  stage('Deploy to Dev') {
                      steps {
                          script {
                              // Deploy to Dev environment using Ansible
                              ansiblePlaybook(
                                  playbook: 'ansible/nextjs-docker-deploy.yml',
                                  inventory: 'ansible/inventories/dev.ini',
                                  extraVars: [
                                      ecr_repository_url: "${ECR_REPOSITORY}",
                                      aws_region: "${AWS_REGION}",
                                      mongodb_uri: "${MONGODB_URI_DEV}",
                                      app_env: 'dev'
                                  ]
                              )
                          }
                      }
                  }
              }
              
              post {
                  always {
                      // Clean up Docker images
                      sh '''
                      docker rmi ${APP_NAME}:${APP_VERSION} || true
                      docker rmi ${ECR_REPOSITORY}:${APP_VERSION} || true
                      docker rmi ${ECR_REPOSITORY}:latest || true
                      '''
                      
                      // Archive artifacts
                      archiveArtifacts artifacts: '.next/**', fingerprint: true
                  }
              }
          }
