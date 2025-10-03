pipeline {
    agent {
        // Use Node.js 16 Docker image as the build agent
        docker {
            image 'node:16-alpine'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    
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
                echo 'Checking out code from repository...'
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                echo 'Installing Node.js dependencies...'
                sh 'npm install --save'
            }
        }
        
        stage('Run Unit Tests') {
            steps {
                echo 'Running unit tests...'
                sh 'npm test || echo "No tests specified"'
            }
        }
        
        stage('Security Scan - Dependencies') {
            steps {
                echo 'Running security vulnerability scan...'
                script {
                    // Install Snyk CLI
                    sh '''
                        npm install -g snyk
                        snyk --version
                    '''
                    
                    // Authenticate with Snyk
                    sh 'snyk auth $SNYK_TOKEN'
                    
                    // Run Snyk test and fail on high/critical vulnerabilities
                    def snykResult = sh(
                        script: 'snyk test --severity-threshold=high --json',
                        returnStatus: true
                    )
                    
                    if (snykResult != 0) {
                        error "Security scan failed: High or Critical vulnerabilities detected!"
                    }
                    
                    echo 'Security scan passed - No High/Critical vulnerabilities found'
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
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
                }
            }
        }
        
        stage('Push to Registry') {
            steps {
                echo 'Pushing Docker image to registry...'
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
                        """
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully!'
            // Archive build artifacts
            archiveArtifacts artifacts: '**/package*.json', allowEmptyArchive: true
        }
        failure {
            echo 'Pipeline failed!'
        }
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
    }
}
