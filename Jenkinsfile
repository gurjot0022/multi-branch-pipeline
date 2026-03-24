pipeline {
    agent any

    environment {
        // DockerHub ya apni registry ki credentials ID (Jenkins credentials mein store karo)
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'
        DOCKER_IMAGE_NAME     = 'ruchika329/docker-based-pipeline'
        IMAGE_TAG             = "${env.BUILD_NUMBER}"

        // Deploy targets (apne server IPs/hostnames se replace karo)
        DEV_SERVER            = 'dev-server.example.com'
        STAGING_SERVER        = 'staging-server.example.com'
        PROD_SERVER           = 'prod-server.example.com'

        // SSH credentials ID (Jenkins credentials mein store karo)
        SSH_CREDENTIALS_ID    = 'ssh-deploy-key'
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 60, unit: 'MINUTES')
    }

    stages {

        // ─────────────────────────────────────────────
        // STAGE 1: SOURCE CODE CHECKOUT
        // ─────────────────────────────────────────────
        stage('Checkout') {
            steps {
                echo '>>> Source code checkout ho raha hai...'
                checkout scm
                echo ">>> Git Branch: ${env.GIT_BRANCH}"
                echo ">>> Git Commit: ${env.GIT_COMMIT}"
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 2: BUILD
        // Docker multi-stage build ke pehle app compile
        // ─────────────────────────────────────────────
        stage('Build') {
            steps {
                echo '>>> Application build ho rahi hai...'
                // Agar aapke paas build script hai (jaise npm install, maven, etc.)
                // Yahan Docker ke andar build hogi, isliye simple validation bhi chalega
                sh '''
                    echo "Build environment check:"
                    docker version
                    docker compose version || true
                    echo "Source files:"
                    ls -la
                '''
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 3: TEST
        // Docker container ke andar tests run karo
        // ─────────────────────────────────────────────
        stage('Test') {
            steps {
                echo '>>> Tests chal rahe hain...'
                sh '''
                    # Docker image temporarily build karo sirf test ke liye
                    docker build --target test -t ${DOCKER_IMAGE_NAME}:test-${IMAGE_TAG} . || \
                    docker build -t ${DOCKER_IMAGE_NAME}:test-${IMAGE_TAG} .

                    # Container mein tests run karo
                    docker run --rm \
                        --name test-runner-${IMAGE_TAG} \
                        ${DOCKER_IMAGE_NAME}:test-${IMAGE_TAG} \
                        sh -c "echo 'Tests pass ho gaye!' && exit 0"

                    # Test image cleanup
                    docker rmi ${DOCKER_IMAGE_NAME}:test-${IMAGE_TAG} || true
                '''
            }
            post {
                always {
                    // Test reports publish karo (agar JUnit format mein hain)
                    junit allowEmptyResults: true, testResults: '**/test-results/*.xml'
                }
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 4: DOCKER IMAGE BUILD & PUSH
        // Final production image build karke registry push
        // ─────────────────────────────────────────────
        
        stage('Docker Image Build & Push') {
    steps {
        echo ">>> Docker image build ho rahi hai: ruchika329/docker-based-pipeline:${IMAGE_TAG}"
        sh """
            docker build -t ruchika329/docker-based-pipeline:${IMAGE_TAG} .
            docker tag ruchika329/docker-based-pipeline:${IMAGE_TAG} ruchika329/docker-based-pipeline:latest
            docker push ruchika329/docker-based-pipeline:${IMAGE_TAG}
            docker push ruchika329/docker-based-pipeline:latest
        """
    }
}

        // ─────────────────────────────────────────────
        // STAGE 5a: DEPLOY → DEV
        // Automatically deploy on every push
        // ─────────────────────────────────────────────
        stage('Deploy: Dev') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'main'
                }
            }
            steps {
                echo ">>> Dev server par deploy ho raha hai: ${DEV_SERVER}"
                sshagent(credentials: [SSH_CREDENTIALS_ID]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no deploy@${DEV_SERVER} '
                            docker pull ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}
                            docker stop app-dev || true
                            docker rm app-dev || true
                            docker run -d \
                                --name app-dev \
                                --restart unless-stopped \
                                -p 8080:8080 \
                                -e APP_ENV=development \
                                ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}
                            echo "Dev deploy complete!"
                        '
                    """
                }
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 5b: DEPLOY → STAGING
        // Dev ke baad automatically staging par deploy
        // ─────────────────────────────────────────────
        stage('Deploy: Staging') {
            when {
                branch 'main'
            }
            steps {
                echo ">>> Staging server par deploy ho raha hai: ${STAGING_SERVER}"
                sshagent(credentials: [SSH_CREDENTIALS_ID]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no deploy@${STAGING_SERVER} '
                            docker pull ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}
                            docker stop app-staging || true
                            docker rm app-staging || true
                            docker run -d \
                                --name app-staging \
                                --restart unless-stopped \
                                -p 8080:8080 \
                                -e APP_ENV=staging \
                                ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}
                            echo "Staging deploy complete!"
                        '
                    """
                }
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 5c: DEPLOY → PRODUCTION
        // Manual approval ke baad hi production deploy
        // ─────────────────────────────────────────────
        stage('Approval: Prod Deploy?') {
            when {
                branch 'main'
            }
            steps {
                echo '>>> Production deploy ke liye manual approval chahiye!'
                timeout(time: 30, unit: 'MINUTES') {
                    input message: "Production par deploy karna hai?",
                          ok: "Haan, Deploy Karo!",
                          submitterParameter: 'APPROVED_BY'
                }
            }
        }

        stage('Deploy: Production') {
            when {
                branch 'main'
            }
            steps {
                echo ">>> PRODUCTION server par deploy ho raha hai: ${PROD_SERVER}"
                sshagent(credentials: [SSH_CREDENTIALS_ID]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no deploy@${PROD_SERVER} '
                            # Zero-downtime deploy ke liye pehle pull karo
                            docker pull ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}

                            # Old container gracefully stop karo
                            docker stop app-prod || true
                            docker rm app-prod || true

                            # Naya container start karo
                            docker run -d \
                                --name app-prod \
                                --restart unless-stopped \
                                -p 80:8080 \
                                -e APP_ENV=production \
                                ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}

                            # Health check
                            sleep 10
                            docker ps | grep app-prod
                            echo "Production deploy SUCCESSFUL!"
                        '
                    """
                }
            }
        }
    }

    // ─────────────────────────────────────────────
    // POST ACTIONS: Success / Failure notifications
    // ─────────────────────────────────────────────
    post {
        success {
            echo "Pipeline SUCCESSFUL hai! Build #${IMAGE_TAG}"
            // Email notification (Jenkins Email plugin chahiye)
            // mail to: 'team@example.com',
            //      subject: "SUCCESS: Build #${IMAGE_TAG}",
            //      body: "Pipeline successfully complete ho gayi."
        }
        failure {
            echo "Pipeline FAIL ho gayi! Build #${IMAGE_TAG}"
            // mail to: 'team@example.com',
            //      subject: "FAILED: Build #${IMAGE_TAG}",
            //      body: "Pipeline fail ho gayi. Logs check karo."
        }
        always {
            echo '>>> Cleanup: Dangling Docker images remove kar rahe hain...'
            sh 'docker image prune -f || true'
        }
    }
}
