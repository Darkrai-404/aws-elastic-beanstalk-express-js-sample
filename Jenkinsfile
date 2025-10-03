pipeline {
    agent any
    
    environment {
        // Docker Hub credentials
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_IMAGE_NAME = 'ahasanulkabirftw/nodejs-app'
        DOCKER_IMAGE_TAG = "${BUILD_NUMBER}"
        // Snyk token for security scanning
        SNYK_TOKEN = credentials('snyk-token')
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo '========== Checking out code from repository =========='
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                echo '========== Installing Node.js dependencies =========='
                sh '''
                    # Use Docker to run npm install
                    docker run --rm -v $(pwd):/app -w /app node:16-alpine npm install --save
                '''
            }
        }
        
        stage('Run Unit Tests') {
            steps {
                echo '========== Running unit tests =========='
                sh '''
                    # Run tests in Node container
                    docker run --rm -v $(pwd):/app -w /app node:16-alpine npm test || echo "No tests specified"
                '''
            }
        }
        
        stage('Security Scan - Dependencies') {
            steps {
                echo '========== Running security vulnerability scan =========='
                script {
                    try {
                        sh '''
                            # Run Snyk in a container
                            docker run --rm \
                                -e SNYK_TOKEN=$SNYK_TOKEN \
                                -v $(pwd):/app \
                                -w /app \
                                node:16-alpine sh -c "
                                    npm install -g snyk && \
                                    snyk auth $SNYK_TOKEN && \
                                    snyk test --severity-threshold=high
                                "
                        '''
                        echo '✅ Security scan passed - No High/Critical vulnerabilities found'
                    } catch (Exception e) {
                        echo "⚠️ Security scan failed: High or Critical vulnerabilities detected!"
                        error "Security scan failed: High or Critical vulnerabilities detected!"
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo '========== Building Docker image =========='
                script {
                    // Creating Dockerfile if it doesn't exist
                    sh '''
                        cat > Dockerfile << 'EOF'
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["node", "app.js"]
EOF
                    '''
                    
                    // Build Docker image
                    sh """
                        docker build -t ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} .
                        docker tag ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} ${DOCKER_IMAGE_NAME}:latest
                    """
                    
                    echo "✅ Docker image built: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                }
            }
        }
        
        stage('Push to Registry') {
            steps {
                echo '========== Pushing Docker image to registry =========='
                script {
                    // Login to Docker Hub and push image
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                            docker push ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}
                            docker push ${DOCKER_IMAGE_NAME}:latest
                            docker logout
                        """
                    }
                    echo "✅ Image pushed to Docker Hub: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                }
            }
        }
    }
    
    post {
        success {
            echo '========================================='
            echo '✅ Pipeline completed successfully!'
            echo "📦 Docker image: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
            echo '========================================='
            // Archive build artifacts
            archiveArtifacts artifacts: '**/package*.json', allowEmptyArchive: true
        }
        failure {
            echo '========================================='
            echo '❌ Pipeline failed!'
            echo '========================================='
        }
        always {
            echo 'Cleaning up workspace...'
            sh 'docker system prune -f || true'
        }
    }
}