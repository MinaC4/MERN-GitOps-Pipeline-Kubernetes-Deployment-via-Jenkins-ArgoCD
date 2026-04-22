pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub')
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        KUBECONFIG = "/var/lib/jenkins/.kube/config"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

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

        stage('Push Images to Docker Hub') {
            steps {
                script {
                    sh '''
                        echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
                    '''

                    sh 'docker push minac4/eshtry-mny-user:${IMAGE_TAG}'
                    sh 'docker push minac4/eshtry-mny-product:${IMAGE_TAG}'
                    sh 'docker push minac4/eshtry-mny-cart:${IMAGE_TAG}'
                    sh 'docker push minac4/eshtry-mny-frontend:${IMAGE_TAG}'
                }
            }
        }

        stage('Deploy to Kubernetes with Helm') {
            steps {
                script {
                    echo " Deploying to Minikube..."

                    sh '''
                        export KUBECONFIG=/var/lib/jenkins/.kube/config

                        echo "Checking Kubernetes connection..."
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
            echo ' Pipeline Success - Application Deployed!'
        }
        failure {
            echo ' Pipeline Failed!'
        }
    }
}
