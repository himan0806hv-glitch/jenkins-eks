// ============================================================
// Jenkins Declarative Pipeline
// CI/CD: Build → Docker → ECR → Deploy to EKS
// ============================================================
//
// PREREQUISITES (configure in Jenkins):
//   1. AWS credentials stored as Jenkins credentials (ID: 'aws-credentials')
//   2. Git credentials if repo is private (ID: 'git-credentials')
//   3. Environment variables set below (AWS_ACCOUNT_ID, AWS_REGION, etc.)
// ============================================================

pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID  = '890608337202'
        AWS_REGION      = 'us-east-1'
        ECR_REPO_NAME   = 'eks-demo'
        EKS_CLUSTER     = 'k8s-cluster'
        ECR_REGISTRY    = "890608337202.dkr.ecr.us-east-1.amazonaws.com/eks-demo"
        IMAGE_TAG       = "${BUILD_NUMBER}"
        FULL_IMAGE      = "${ECR_REGISTRY}/${ECR_REPO_NAME}:${IMAGE_TAG}"
    }

    stages {
        stage('Checkout') {
            steps {
                echo '📥 Pulling source code from Git...'
                checkout scm
            }
        }

        stage('Build with Maven') {
            steps {
                echo '🔨 Building application with Maven...'
                sh 'mvn clean package -DskipTests -B'
            }
        }

        stage('Run Tests') {
            steps {
                echo '🧪 Running unit tests...'
                sh 'mvn test -B'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🐳 Building Docker image: ${FULL_IMAGE}"
                sh "docker build -t ${FULL_IMAGE} ."
                sh "docker tag ${FULL_IMAGE} ${ECR_REGISTRY}/${ECR_REPO_NAME}:latest"
            }
        }

        stage('Push to ECR') {
            steps {
                echo '📤 Pushing Docker image to Amazon ECR...'
                sh """
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    docker push ${FULL_IMAGE}
                    docker push ${ECR_REGISTRY}/${ECR_REPO_NAME}:latest
                """
            }
        }

        stage('Deploy to EKS') {
            steps {
                echo '🚀 Deploying to Amazon EKS...'
                sh """
                    aws eks update-kubeconfig \
                        --name ${k8s-cluster} \
                        --region ${us-east-1}

                    sed -i 's|IMAGE_PLACEHOLDER|${FULL_IMAGE}|g' k8s/deployment.yaml

                    kubectl apply -f k8s/deployment.yaml
                    kubectl apply -f k8s/service.yaml

                    kubectl rollout status deployment/devops-demo \
                        --timeout=120s
                """
            }
        }

        stage('Verify') {
            steps {
                echo '✅ Verifying deployment...'
                sh """
                    echo "─── Pods ───"
                    kubectl get pods -l app=devops-demo
                    echo ""
                    echo "─── Service ───"
                    kubectl get svc devops-demo-service
                    echo ""
                    echo "─── Deployment ───"
                    kubectl get deployment devops-demo
                """
            }
        }
    }

    post {
        success {
            echo '🎉 Pipeline completed successfully! Application deployed to EKS.'
        }
        failure {
            echo '❌ Pipeline failed. Check logs above for details.'
        }
        always {
            sh "docker rmi ${FULL_IMAGE} || true"
            sh "docker rmi ${ECR_REGISTRY}/${ECR_REPO_NAME}:latest || true"
        }
    }
}
