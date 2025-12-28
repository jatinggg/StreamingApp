pipeline {
    agent any
    
    environment {
        AWS_REGION = 'eu-west-2'
        AWS_ACCOUNT_ID = credentials('jatin_aws_account_id')
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        S3_BUCKET = credentials('jatin-streamingapp-28122025')
        IMAGE_TAG = "${BUILD_NUMBER}"
        GIT_COMMIT_SHORT = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1'
            }
        }
        
        stage('Build Docker Images') {
            //parallel {
                stage('Build Auth Service') {
                    steps {
                        script {
                            sh """
                                cd backend
                                docker build -t ${ecrRegistry}/streamingapp/auth:${IMAGE_TAG} \
                                -f authService/Dockerfile \
                                .
                            """
                        }
                    }
                }
                stage('Build Streaming Service') {
                    steps {
                        script {
                            sh """
                                docker build -t ${ECR_REGISTRY}/streamingapp/streaming:${IMAGE_TAG} \
                                  -t ${ECR_REGISTRY}/streamingapp/streaming:latest \
                                  -f backend/streamingService/Dockerfile backend/
                            """
                        }
                    }
                }
                stage('Build Admin Service') {
                    steps {
                        script {
                            sh """
                                docker build -t ${ECR_REGISTRY}/streamingapp/admin:${IMAGE_TAG} \
                                  -t ${ECR_REGISTRY}/streamingapp/admin:latest \
                                  -f backend/adminService/Dockerfile backend/
                            """
                        }
                    }
                }
                stage('Build Chat Service') {
                    steps {
                        script {
                            sh """
                                docker build -t ${ECR_REGISTRY}/streamingapp/chat:${IMAGE_TAG} \
                                  -t ${ECR_REGISTRY}/streamingapp/chat:latest \
                                  -f backend/chatService/Dockerfile backend/
                            """
                        }
                    }
                }
                stage('Build Frontend') {
                    steps {
                        script {
                            sh """
                                docker build -t ${ECR_REGISTRY}/streamingapp/frontend:${IMAGE_TAG} \
                                  -t ${ECR_REGISTRY}/streamingapp/frontend:latest \
                                  --build-arg REACT_APP_AUTH_API_URL=http://api.streamingapp.com/auth \
                                  --build-arg REACT_APP_STREAMING_API_URL=http://api.streamingapp.com/streaming \
                                  --build-arg REACT_APP_ADMIN_API_URL=http://api.streamingapp.com/admin \
                                  --build-arg REACT_APP_CHAT_API_URL=http://api.streamingapp.com/chat \
                                  frontend/
                            """
                        }
                    }
                }
            //}
        }
        
        stage('Push to ECR') {
            steps {
                script {
                    withAWS(credentials: 'JATIN_AWS_CRED', region: "${AWS_REGION}") {
                        sh """
                            aws ecr get-login-password --region ${AWS_REGION} | \
                            docker login --username AWS --password-stdin ${ECR_REGISTRY}
                            
                            docker push ${ECR_REGISTRY}/streamingapp/auth:${IMAGE_TAG}
                            docker push ${ECR_REGISTRY}/streamingapp/auth:latest
                            
                            docker push ${ECR_REGISTRY}/streamingapp/streaming:${IMAGE_TAG}
                            docker push ${ECR_REGISTRY}/streamingapp/streaming:latest
                            
                            docker push ${ECR_REGISTRY}/streamingapp/admin:${IMAGE_TAG}
                            docker push ${ECR_REGISTRY}/streamingapp/admin:latest
                            
                            docker push ${ECR_REGISTRY}/streamingapp/chat:${IMAGE_TAG}
                            docker push ${ECR_REGISTRY}/streamingapp/chat:latest
                            
                            docker push ${ECR_REGISTRY}/streamingapp/frontend:${IMAGE_TAG}
                            docker push ${ECR_REGISTRY}/streamingapp/frontend:latest
                        """
                    }
                }
            }
        }
        
        /*
        stage('Update Helm Values') {
            steps {
                script {
                    sh """
                        sed -i 's|tag:.*|tag: "${IMAGE_TAG}"|g' helm/streamingapp/values.yaml
                        git config user.email "jenkins@streamingapp.com"
                        git config user.name "Jenkins CI"
                        git add helm/streamingapp/values.yaml
                        git commit -m "Update image tags to ${IMAGE_TAG}" || true
                    """
                }
            }
        }
        
        
        stage('Deploy to EKS') {
            steps {
                script {
                    withAWS(credentials: 'JATIN_AWS_CRED', region: "${AWS_REGION}") {
                        sh """
                            aws eks update-kubeconfig --name streamingapp-cluster --region ${AWS_REGION}
                            helm upgrade --install streamingapp ./helm/streamingapp \
                              --namespace production \
                              --create-namespace \
                              --set image.tag=${IMAGE_TAG} \
                              --wait --timeout 10m
                        """
                    }
                }
            }
        }
        */
    }
    
    post {
        success {
            echo 'Pipeline succeeded! Application deployed successfully.'
            // SNS notification will be added in ChatOps section
        }
        failure {
            echo 'Pipeline failed! Check logs for details.'
            // SNS notification will be added in ChatOps section
        }
        always {
            cleanWs()
        }
    }
}

