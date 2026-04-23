pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub')
        IMAGE_TAG = "${env.BUILD_NUMBER}"
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

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh 'docker build -t minac4/eshtry-mny-user:${IMAGE_TAG} ./User'
                sh 'docker build -t minac4/eshtry-mny-product:${IMAGE_TAG} ./Product'
                sh 'docker build -t minac4/eshtry-mny-cart:${IMAGE_TAG} ./Cart'
                sh 'docker build -t minac4/eshtry-mny-frontend:${IMAGE_TAG} ./front-end'
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
                sh '''
                    docker run --rm aquasec/trivy image minac4/eshtry-mny-user:${IMAGE_TAG} || true
                    docker run --rm aquasec/trivy image minac4/eshtry-mny-product:${IMAGE_TAG} || true
                    docker run --rm aquasec/trivy image minac4/eshtry-mny-cart:${IMAGE_TAG} || true
                    docker run --rm aquasec/trivy image minac4/eshtry-mny-frontend:${IMAGE_TAG} || true
                '''
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'

                sh 'docker push minac4/eshtry-mny-user:${IMAGE_TAG}'
                sh 'docker push minac4/eshtry-mny-product:${IMAGE_TAG}'
                sh 'docker push minac4/eshtry-mny-cart:${IMAGE_TAG}'
                sh 'docker push minac4/eshtry-mny-frontend:${IMAGE_TAG}'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    export KUBECONFIG=/var/lib/jenkins/.kube/config

                    kubectl apply -f k8s/

                    kubectl set image deployment/user user=minac4/eshtry-mny-user:${IMAGE_TAG}
                    kubectl set image deployment/product product=minac4/eshtry-mny-product:${IMAGE_TAG}
                    kubectl set image deployment/cart cart=minac4/eshtry-mny-cart:${IMAGE_TAG}
                    kubectl set image deployment/frontend frontend=minac4/eshtry-mny-frontend:${IMAGE_TAG}

                    kubectl get svc
                '''
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
