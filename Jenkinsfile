pipeline {
    agent any

    environment {
        AWS_REGION       = 'eu-west-2'
        AWS_ACCOUNT_ID   = credentials('jatin_aws_account_id')
        ECR_REGISTRY     = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        APP_PREFIX       = 'jatin-streamingapp'
        S3_BUCKET        = credentials('jatin-streamingapp-28122025')
        IMAGE_TAG        = "${BUILD_NUMBER}"
        EKS_CLUSTER_NAME = 'jatin-streamingapp-cluster'
        GIT_COMMIT_SHORT = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1'
            }
        }

        stage('Build Docker Images in Parallel') {
            parallel {
                stage('Build Auth Service') {
                    steps {
                        script {
                            sh """
                                echo "Building Auth Service..."
                                docker build -t ${ECR_REGISTRY}/${APP_PREFIX}/auth:${IMAGE_TAG} \
                                  -t ${ECR_REGISTRY}/${APP_PREFIX}/auth:latest \
                                  -f backend/authService/Dockerfile \
                                  backend/
                            """
                        }
                    }
                }

                stage('Build Streaming Service') {
                    steps {
                        script {
                            sh """
                                echo "Building Streaming Service..."
                                docker build -t ${ECR_REGISTRY}/${APP_PREFIX}/streaming:${IMAGE_TAG} \
                                  -t ${ECR_REGISTRY}/${APP_PREFIX}/streaming:latest \
                                  -f backend/streamingService/Dockerfile \
                                  backend/
                            """
                        }
                    }
                }

                stage('Build Admin Service') {
                    steps {
                        script {
                            sh """
                                echo "Building Admin Service..."
                                docker build -t ${ECR_REGISTRY}/${APP_PREFIX}/admin:${IMAGE_TAG} \
                                  -t ${ECR_REGISTRY}/${APP_PREFIX}/admin:latest \
                                  -f backend/adminService/Dockerfile \
                                  backend/
                            """
                        }
                    }
                }

                stage('Build Chat Service') {
                    steps {
                        script {
                            sh """
                                echo "Building Chat Service..."
                                docker build -t ${ECR_REGISTRY}/${APP_PREFIX}/chat:${IMAGE_TAG} \
                                  -t ${ECR_REGISTRY}/${APP_PREFIX}/chat:latest \
                                  -f backend/chatService/Dockerfile \
                                  backend/
                            """
                        }
                    }
                }

                stage('Build Frontend') {
                    steps {
                        script {
                            sh """
                                echo "Building Frontend..."
                                docker build -t ${ECR_REGISTRY}/${APP_PREFIX}/frontend:${IMAGE_TAG} \
                                  -t ${ECR_REGISTRY}/${APP_PREFIX}/frontend:latest \
                                  --build-arg REACT_APP_AUTH_API_URL=http://api.streamingapp.com/auth \
                                  --build-arg REACT_APP_STREAMING_API_URL=http://api.streamingapp.com/streaming \
                                  --build-arg REACT_APP_ADMIN_API_URL=http://api.streamingapp.com/admin \
                                  --build-arg REACT_APP_CHAT_API_URL=http://api.streamingapp.com/chat \
                                  frontend/
                            """
                        }
                    }
                }
            }
        }

        stage('Verify Images Built') {
            steps {
                script {
                    sh """
                        echo "=== Verifying built images ==="
                        docker images | grep jatin-streamingapp || echo "No images found"
                        echo "=== Verification complete ==="
                    """
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    withAWS(credentials: 'JATIN_AWS_CRED', region: "${AWS_REGION}") {
                        sh """
                            aws ecr get-login-password --region ${AWS_REGION} | \
                            docker login --username AWS --password-stdin ${ECR_REGISTRY}
                            
                            echo "Pushing auth service..."
                            docker push ${ECR_REGISTRY}/${APP_PREFIX}/auth:${IMAGE_TAG}
                            docker push ${ECR_REGISTRY}/${APP_PREFIX}/auth:latest
                            
                            echo "Pushing streaming service..."
                            docker push ${ECR_REGISTRY}/${APP_PREFIX}/streaming:${IMAGE_TAG}
                            docker push ${ECR_REGISTRY}/${APP_PREFIX}/streaming:latest
                            
                            echo "Pushing admin service..."
                            docker push ${ECR_REGISTRY}/${APP_PREFIX}/admin:${IMAGE_TAG}
                            docker push ${ECR_REGISTRY}/${APP_PREFIX}/admin:latest
                            
                            echo "Pushing chat service..."
                            docker push ${ECR_REGISTRY}/${APP_PREFIX}/chat:${IMAGE_TAG}
                            docker push ${ECR_REGISTRY}/${APP_PREFIX}/chat:latest
                            
                            echo "Pushing frontend..."
                            docker push ${ECR_REGISTRY}/${APP_PREFIX}/frontend:${IMAGE_TAG}
                            docker push ${ECR_REGISTRY}/${APP_PREFIX}/frontend:latest
                        """
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    withAWS(credentials: 'JATIN_AWS_CRED', region: "${AWS_REGION}") {
                        sh """
                            echo "Configuring kubectl for EKS cluster..."
                            aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}
                            
                            echo "Deploying application with Helm..."
                            helm upgrade --install streamingapp ./helm/streamingapp \
                              --namespace production \
                              --create-namespace \
                              --set image.tag=${IMAGE_TAG} \
                              --wait --timeout 10m
                            
                            echo "Verifying deployment..."
                            kubectl get pods -n production
                            kubectl get svc -n production
                            
                            echo "Getting frontend URL..."
                            kubectl get svc frontend -n production -o wide
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                echo 'Pipeline succeeded!'
                def topicArn = "arn:aws:sns:${AWS_REGION}:${AWS_ACCOUNT_ID}:jatin-streamingapp-notifications"
                
                withAWS(credentials: 'JATIN_AWS_CRED', region: "${AWS_REGION}") {
                    sh """
                        aws sns publish \
                        --topic-arn ${topicArn} \
                        --message "✅ Deployment Success: Build #${env.BUILD_NUMBER} for ${env.JOB_NAME} deployed to EKS." \
                        --subject "Jenkins Build Success"
                    """
                }
            }
        }
        failure {
            script {
                echo 'Pipeline failed!'
                def topicArn = "arn:aws:sns:${AWS_REGION}:${AWS_ACCOUNT_ID}:jatin-streamingapp-notifications"

                withAWS(credentials: 'JATIN_AWS_CRED', region: "${AWS_REGION}") {
                    sh """
                        aws sns publish \
                        --topic-arn ${topicArn} \
                        --message "❌ Deployment Failed: Build #${env.BUILD_NUMBER} for ${env.JOB_NAME}. Check logs." \
                        --subject "Jenkins Build Failure"
                    """
                }
            }
        }
        always {
            cleanWs()
        }
    }
}