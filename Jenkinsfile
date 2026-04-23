pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub')
        SONAR_TOKEN = credentials('sonar-token')
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        KUBECONFIG = "/var/lib/jenkins/.kube/config"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        // ==================== SONARQUBE ====================
        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv('sonarqube') {
                        sh '''
                            sonar-scanner \
                              -Dsonar.projectKey=eshtry-mny \
                              -Dsonar.sources=. \
                              -Dsonar.host.url=http://localhost:9000 \
                              -Dsonar.login=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ==================== BUILD ====================
        stage('Build Docker Images') {
            steps {
                script {
                    sh 'docker build -t minac4/eshtry-mny-user:${IMAGE_TAG} ./User'
                    sh 'docker build -t minac4/eshtry-mny-product:${IMAGE_TAG} ./Product'
                    sh 'docker build -t minac4/eshtry-mny-cart:${IMAGE_TAG} ./Cart'
                    sh 'docker build -t minac4/eshtry-mny-frontend:${IMAGE_TAG} ./front-end'
                }
            }
        }

        // ==================== SECURITY ====================
        stage('Security: Dependency Audit') {
            steps {
                echo 'Running npm audit on all services...'
                sh 'cd User && npm audit || true'
                sh 'cd Product && npm audit || true'
                sh 'cd Cart && npm audit || true'
                sh 'cd front-end && npm audit || true'
            }
        }

        stage('Security: Docker Scan (Trivy)') {
            steps {
                echo 'Scanning Docker images with Trivy...'
                sh '''
                    docker run --rm aquasec/trivy image minac4/eshtry-mny-user:${IMAGE_TAG} --severity HIGH,CRITICAL --exit-code 0 || true
                    docker run --rm aquasec/trivy image minac4/eshtry-mny-product:${IMAGE_TAG} --severity HIGH,CRITICAL --exit-code 0 || true
                    docker run --rm aquasec/trivy image minac4/eshtry-mny-cart:${IMAGE_TAG} --severity HIGH,CRITICAL --exit-code 0 || true
                    docker run --rm aquasec/trivy image minac4/eshtry-mny-frontend:${IMAGE_TAG} --severity HIGH,CRITICAL --exit-code 0 || true
                '''
            }
        }

        // ==================== PUSH ====================
        stage('Push Images to Docker Hub') {
            steps {
                script {
                    sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                    sh 'docker push minac4/eshtry-mny-user:${IMAGE_TAG}'
                    sh 'docker push minac4/eshtry-mny-product:${IMAGE_TAG}'
                    sh 'docker push minac4/eshtry-mny-cart:${IMAGE_TAG}'
                    sh 'docker push minac4/eshtry-mny-frontend:${IMAGE_TAG}'
                }
            }
        }

        // ==================== DEPLOY ====================
        stage('Deploy to Kubernetes with Helm') {
            steps {
                script {
                    echo "Deploying to Minikube..."
                    sh '''
                        export KUBECONFIG=/var/lib/jenkins/.kube/config
                        kubectl get nodes
                        cd eshtry-mny
                        helm upgrade --install eshtry-mny . \
                          --set images.user=minac4/eshtry-mny-user:${IMAGE_TAG} \
                          --set images.product=minac4/eshtry-mny-product:${IMAGE_TAG} \
                          --set images.cart=minac4/eshtry-mny-cart:${IMAGE_TAG} \
                          --set images.frontend=minac4/eshtry-mny-frontend:${IMAGE_TAG}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline Success - Deployed with SonarQube Quality Gate!'
        }
        failure {
            echo 'Pipeline Failed!'
        }
    }
}
