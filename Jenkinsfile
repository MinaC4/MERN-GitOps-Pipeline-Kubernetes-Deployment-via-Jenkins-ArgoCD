pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub')
        IMAGE_TAG = "${BUILD_NUMBER}"
        KUBECONFIG = "/var/lib/jenkins/.kube/config"
        SONAR_SCANNER = "/opt/sonar-scanner/bin/sonar-scanner"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh """
                            ${SONAR_SCANNER} \
                            -Dsonar.projectKey=eshtry-mny \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=http://localhost:9000 \
                            -Dsonar.login=$SONAR_TOKEN
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline failed بسبب Quality Gate: ${qg.status}"
                        }
                    }
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh "docker build -t minac4/eshtry-mny-user:${IMAGE_TAG} ./User"
                sh "docker build -t minac4/eshtry-mny-product:${IMAGE_TAG} ./Product"
                sh "docker build -t minac4/eshtry-mny-cart:${IMAGE_TAG} ./Cart"
                sh "docker build -t minac4/eshtry-mny-frontend:${IMAGE_TAG} ./front-end"
            }
        }

        stage('Security: Dependency Audit') {
            steps {
                sh 'cd User && npm audit || true'
                sh 'cd Product && npm audit || true'
                sh 'cd Cart && npm audit || true'
                sh 'cd front-end && npm audit || true'
            }
        }

        stage('Security: Docker Scan (Trivy)') {
            steps {
                sh """
                    docker run --rm aquasec/trivy image minac4/eshtry-mny-user:${IMAGE_TAG} --severity HIGH,CRITICAL || true
                    docker run --rm aquasec/trivy image minac4/eshtry-mny-product:${IMAGE_TAG} --severity HIGH,CRITICAL || true
                    docker run --rm aquasec/trivy image minac4/eshtry-mny-cart:${IMAGE_TAG} --severity HIGH,CRITICAL || true
                    docker run --rm aquasec/trivy image minac4/eshtry-mny-frontend:${IMAGE_TAG} --severity HIGH,CRITICAL || true
                """
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                sh "docker push minac4/eshtry-mny-user:${IMAGE_TAG}"
                sh "docker push minac4/eshtry-mny-product:${IMAGE_TAG}"
                sh "docker push minac4/eshtry-mny-cart:${IMAGE_TAG}"
                sh "docker push minac4/eshtry-mny-frontend:${IMAGE_TAG}"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                    export KUBECONFIG=/var/lib/jenkins/.kube/config
                    kubectl get nodes
                    cd eshtry-mny
                    helm upgrade --install eshtry-mny . \
                      --set images.user=minac4/eshtry-mny-user:${IMAGE_TAG} \
                      --set images.product=minac4/eshtry-mny-product:${IMAGE_TAG} \
                      --set images.cart=minac4/eshtry-mny-cart:${IMAGE_TAG} \
                      --set images.frontend=minac4/eshtry-mny-frontend:${IMAGE_TAG}
                """
            }
        }
    }

    post {
        success {
            echo 'Pipeline Success'
        }
        failure {
            echo 'Pipeline Failed'
        }
    }
}
