
pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'manoj1812'
        IMAGE_NAME = 'employee-service'
        IMAGE_TAG = "${BUILD_NUMBER}"
        K8S_NAMESPACE = 'production'
        AWS_REGION = 'us-east-1'
        EKS_CLUSTER = 'production-eks-demo'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            steps {
                sh 'chmod +x mvnw'
                sh './mvnw clean verify'
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t ${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} \
                        ${DOCKERHUB_USER}/${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Docker Hub Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_TOKEN'
                )]) {
                    sh '''
                        set -e
                        echo "$DOCKER_TOKEN" | docker login \
                            -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${DOCKERHUB_USER}/${IMAGE_NAME}:latest
                        docker logout
                    '''
                }
            }
        }

        stage('Configure EKS Access') {
            steps {
                sh '''
                    aws eks update-kubeconfig \
                        --region ${AWS_REGION} \
                        --name ${EKS_CLUSTER}
                    kubectl auth can-i patch deployment/employee-app \
                        -n ${K8S_NAMESPACE}
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    kubectl set image deployment/employee-app \
                        employee-app=${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} \
                        -n ${K8S_NAMESPACE}

                    kubectl rollout status deployment/employee-app \
                        -n ${K8S_NAMESPACE} --timeout=180s
                '''
            }
        }
    }

    post {
        success {
            echo 'Build, tests, Docker push and EKS deployment completed!'
        }
        failure {
            echo 'Pipeline failed. Check Console Output.'
        }
    }
}
