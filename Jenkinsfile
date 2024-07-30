pipeline {
    agent any
    environment {
        DOCKERHUB_USERNAME = "sharukdoc"
        APP_NAME = "insignia-app"
        IMAGE_TAG = "${BUILD_NUMBER}"
        IMAGE_NAME = "${DOCKERHUB_USERNAME}/${APP_NAME}"
        REGISTRY_CREDS = "Dockerhub"
        EKS_CLUSTER_NAME = "insignia-eks-cluster"
        AWS_REGION = "us-west-2"
    }
    stages {
        stage('Cleanup workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout SCM') {
            steps {
                git credentialsId: 'github',
                    url: 'https://github.com/your-username/Insignia-app.git',
                    branch: 'main'
            }
        }
        stage('Create EKS Cluster') {
            steps {
                script {
                    sh "eksctl create cluster -f cluster-config.yaml"
                }
            }
        }
        stage('Deploy Prometheus and Grafana') {
            steps {
                script {
                    sh """
                    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
                    helm repo update
                    helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
                    """
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    docker_image = docker.build "${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('', REGISTRY_CREDS) {
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }
        }
        stage('Update Kubernetes deployment file') {
            steps {
                script {
                    sh """
                    sed -i 's|${IMAGE_NAME}:.*|${IMAGE_NAME}:${IMAGE_TAG}|' kubernetes/deployment.yaml
                    """
                }
            }
        }
        stage('Deploy to EKS') {
            steps {
                script {
                    sh """
                    kubectl apply -f kubernetes/secret-provider.yaml
                    kubectl apply -f kubernetes/deployment.yaml
                    kubectl apply -f kubernetes/service.yaml
                    kubectl apply -f kubernetes/service-monitor.yaml
                    """
                }
            }
        }
        stage('Update Secret') {
            steps {
                script {
                    withCredentials([aws(credentialsId: 'aws-credentials', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh """
                        aws secretsmanager update-secret --secret-id insignia-app-secret --secret-string '{"key":"value"}' --region us-west-2
                        """
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                sh "eksctl delete cluster --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}"
            }
        }
    }
}