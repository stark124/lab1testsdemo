
pipeline {

    agent any

    environment {

        AWS_REGION = 'ap-south-1'

        AWS_ACCOUNT_ID = '992382458064'

        ECR_REPO = 'lab1test'

        EKS_CLUSTER = 'Lab1_Test'

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        IMAGE_TAG = "${BUILD_NUMBER}"

        IMAGE_URI = "${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify Tools') {
            steps {
                sh '''
                    echo "===== TOOL VERSION ====="

                    git --version
                    docker --version
                    aws --version
                    kubectl version --client

                    echo "===== CONFIGURATION ====="

                    echo "AWS Region     : ${AWS_REGION}"
                    echo "AWS Account    : ${AWS_ACCOUNT_ID}"
                    echo "ECR Repository : ${ECR_REPO}"
                    echo "EKS Cluster    : ${EKS_CLUSTER}"
                    echo "Image URI      : ${IMAGE_URI}"
                '''
            }
        }

        stage('AWS Identity') {
            steps {
                sh '''
                    echo "===== AWS IDENTITY ====="

                    aws sts get-caller-identity
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "===== BUILD DOCKER IMAGE ====="

                    docker build -t ${ECR_REPO}:${IMAGE_TAG} .

                    docker images | grep ${ECR_REPO}
                '''
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                    echo "===== LOGIN TO ECR ====="

                    aws ecr get-login-password \
                        --region ${AWS_REGION} | \
                    docker login \
                        --username AWS \
                        --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    echo "===== TAG IMAGE ====="

                    docker tag \
                        ${ECR_REPO}:${IMAGE_TAG} \
                        ${IMAGE_URI}

                    echo "===== PUSH IMAGE ====="

                    docker push ${IMAGE_URI}
                '''
            }
        }

        stage('Configure EKS') {
            steps {
                sh '''
                    echo "===== UPDATE EKS KUBECONFIG ====="

                    aws eks update-kubeconfig \
                        --region ${AWS_REGION} \
                        --name ${EKS_CLUSTER}

                    echo "===== CHECK EKS CONNECTION ====="

                    kubectl cluster-info

                    echo "===== CHECK NODES ====="

                    kubectl get nodes
                '''
            }
        }

        stage('Deploy Application') {
            steps {
                sh '''
                    echo "===== DEPLOY APPLICATION ====="

                    sed "s|IMAGE_URI|${IMAGE_URI}|g" \
                        deployment.yaml | \
                    kubectl apply -f -

                    kubectl apply -f service.yaml
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "===== WAIT FOR ROLLOUT ====="

                    kubectl rollout status \
                        deployment/jenkins-nginx \
                        --timeout=180s

                    echo "===== DEPLOYMENT ====="

                    kubectl get deployment jenkins-nginx

                    echo "===== PODS ====="

                    kubectl get pods \
                        -l app=jenkins-nginx \
                        -o wide

                    echo "===== SERVICE ====="

                    kubectl get service jenkins-nginx-service
                '''
            }
        }
    }

    post {

        success {
            echo '''
            ========================================
            Jenkins EKS Deployment SUCCESSFUL
            ========================================
            '''

            echo "Application : lab1testdemo"
            echo "ECR Repo    : ${ECR_REPO}"
            echo "EKS Cluster : ${EKS_CLUSTER}"
            echo "AWS Region  : ${AWS_REGION}"
            echo "Image       : ${IMAGE_URI}"
        }

        failure {
            echo '''
            ========================================
            Jenkins EKS Deployment FAILED
            ========================================
            Check Jenkins Console Output.
            '''
        }
    }
}
