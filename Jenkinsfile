pipeline {
    agent any

    environment {
        APP_NAME        = 'django-notes-app'
        AWS_REGION      = 'us-east-1'
        ECR_REGISTRY    = 'YOUR_AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com'   // ← Yeh change karna hai
        IMAGE_TAG       = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
        DOCKER_IMAGE    = "${ECR_REGISTRY}/${APP_NAME}:${IMAGE_TAG}"
    }

    stages {
        stage('Branch Info') {
            steps {
                echo "🌿 Branch: ${env.BRANCH_NAME}"
                echo "🔨 Build Number: ${env.BUILD_NUMBER}"
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
                echo '✅ Git se source code pull ho gaya'
            }
        }

        stage('Build') {
            steps {
                echo '✅ App compile / artifacts ready'
                sh 'pip install -r requirements.txt'
                sh 'python manage.py collectstatic --noinput'
            }
        }

        stage('Test') {
            steps {
                echo '✅ Unit + integration tests run'
                sh 'python manage.py test'
            }
        }

        stage('Docker image build') {
            when {
                anyOf {
                    branch 'dev'
                    branch 'feature/*'
                    branch 'main'
                    branch 'prod'
                }
            }
            steps {
                echo '✅ Docker image build + ECR push'
                script {
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        docker build -t ${DOCKER_IMAGE} .
                        docker push ${DOCKER_IMAGE}
                    """
                }
            }
        }

        // ================== Deploy Section (Photo jaisa) ==================
        stage('Deploy: Dev') {
            when { anyOf { branch 'dev'; branch 'feature/*' } }
            steps {
                echo '🚀 Deploy: Dev - Auto deploy, fast feedback'
                sh """
                    aws ecs update-service --cluster dev-cluster-name \
                    --service dev-service-name --force-new-deployment --region ${AWS_REGION}
                """
            }
            post {
                success { echo '✅ Success + Notify (Dev)' }
            }
        }

        stage('Deploy: Staging') {
            when { branch 'main' }
            steps {
                input message: 'QA testing, approval gate - Approve for Staging?', ok: 'Approve'
                echo '🚀 Deploy: Staging - QA testing, approval gate'
                sh """
                    aws ecs update-service --cluster staging-cluster-name \
                    --service staging-service-name --force-new-deployment --region ${AWS_REGION}
                """
            }
            post {
                success { echo '✅ Success + Notify (Staging)' }
            }
        }

        stage('Deploy: Prod') {
            when { branch 'prod' }
            steps {
                input message: 'Manual approval required - Approve for Production?', ok: 'Deploy to Prod'
                echo '🚀 Deploy: Prod - Manual approval required'
                sh """
                    aws ecs update-service --cluster prod-cluster-name \
                    --service prod-service-name --force-new-deployment --region ${AWS_REGION}
                """
            }
            post {
                success { echo '✅ Success + Notify (Prod)' }
            }
        }
    }

    post {
        failure {
            echo '❌ Koi bhi stage fail hone par → Fail + Alert'
            // yahan Slack/Email alert add kar sakte ho
        }
        always {
            echo "Pipeline complete for branch: ${env.BRANCH_NAME}"
        }
    }
}
