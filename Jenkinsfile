pipeline {
agent any


environment {
    AWS_ACCOUNT_ID  = '890608337202'
    AWS_REGION      = 'us-east-1'
    ECR_REPO_NAME   = 'eks-demo'
    EKS_CLUSTER     = 'k8s-cluster'

    ECR_REGISTRY    = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
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
                    --name ${EKS_CLUSTER} \
                    --region ${AWS_REGION}

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
                kubectl get pods -l app=devops-demo
                kubectl get svc devops-demo-service
                kubectl get deployment devops-demo
            """
        }
    }
}

post {
    success {
        echo '🎉 Pipeline completed successfully!'
    }
    failure {
        echo '❌ Pipeline failed. Check logs above.'
    }
    always {
        sh "docker rmi ${FULL_IMAGE} || true"
        sh "docker rmi ${ECR_REGISTRY}/${ECR_REPO_NAME}:latest || true"
    }
}


}
