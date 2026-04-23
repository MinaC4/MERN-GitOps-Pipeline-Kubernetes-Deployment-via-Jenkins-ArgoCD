pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub')
        SONAR_TOKEN = credentials('sonar-token')
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        KUBECONFIG = "/var/lib/jenkins/.kube/config"
        SONAR_SCANNER = "/opt/sonar-scanner/bin/sonar-scanner"
        SONAR_HOST = "http://192.168.X.X:9000"
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
                withSonarQubeEnv('sonarqube') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            ${SONAR_SCANNER} \
                            -Dsonar.projectKey=eshtry-mny \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=${SONAR_HOST} \
                            -Dsonar.login=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ==================== BUILD ====================
        stage('Build Docker Images') {
            steps {
                script {
                    def services = ["User", "Product", "Cart", "front-end"]

                    for (s in services) {
                        sh "docker build -t minac4/eshtry-mny-${s.toLowerCase()}:${IMAGE_TAG} ./${s}"
                    }
                }
            }
        }

        // ==================== SECURITY ====================
        stage('Security: Dependency Audit') {
            steps {
                sh '''
                    cd User && npm audit || true
                    cd ../Product && npm audit || true
                    cd ../Cart && npm audit || true
                    cd ../front-end && npm audit || true
                '''
            }
        }

        stage('Security: Docker Scan (Trivy)') {
            steps {
                script {
                    def images = ["user", "product", "cart", "frontend"]

                    for (img in images) {
                        sh """
                            docker run --rm aquasec/trivy image minac4/eshtry-mny-${img}:${IMAGE_TAG} \
                            --severity HIGH,CRITICAL || true
                        """
                    }
                }
            }
        }

        // ==================== PUSH ====================
        stage('Push Images to Docker Hub') {
            steps {
                sh '''
                    echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin

                    docker push minac4/eshtry-mny-user:${IMAGE_TAG}
                    docker push minac4/eshtry-mny-product:${IMAGE_TAG}
                    docker push minac4/eshtry-mny-cart:${IMAGE_TAG}
                    docker push minac4/eshtry-mny-frontend:${IMAGE_TAG}
                '''
            }
        }

        // ==================== DEPLOY ====================
        stage('Deploy to Kubernetes with Helm') {
            steps {
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

    post {
        success {
            echo 'Pipeline Success - DevSecOps Flow Completed'
        }
        failure {
            echo 'Pipeline Failed!'
        }
    }
}
