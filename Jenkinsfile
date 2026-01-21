pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'ajiththomas10'
        IMAGE_TAG = "${BUILD_NUMBER}"
        KUBERNETES_SERVER = 'https://kubernetes.default.svc'  
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/ajiththomas333/k85.git',
                    credentialsId: 'git'
            }
        }

        stage('Install Dependencies & Test Backend') {
            steps {
                dir('backend') {
                    bat 'npm install'
                    bat 'npm test'
                }
            }
        }

        stage('Install Dependencies & Test Frontend') {
            steps {
                dir('frontend') {
                    bat 'npm install'
                    bat 'npm test -- --coverage --watchAll=false'
                }
            }
        }

        stage('Build & Push Docker Images') {
            parallel {
                stage('Backend Image') {
                    steps {
                        script {
                            dir('backend') {
                                def backendImage = docker.build("${DOCKER_REGISTRY}/mern-backend:${IMAGE_TAG}")
                                docker.withRegistry('https://registry.hub.docker.com', 'dockercred') {
                                    backendImage.push()
                                    backendImage.push('latest')
                                }
                            }
                        }
                    }
                }
                stage('Frontend Image') {
                    steps {
                        script {
                            dir('frontend') {
                                def frontendImage = docker.build("${DOCKER_REGISTRY}/mern-frontend:${IMAGE_TAG}")
                                docker.withRegistry('https://registry.hub.docker.com', 'dockercred') {
                                    frontendImage.push()
                                    frontendImage.push('latest')
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Verify Kubernetes Connectivity') {
            steps {
                script {
                    echo '🔍 Verifying Kubernetes API (In-cluster ServiceAccount)...'
                    bat 'kubectl cluster-info'
                    bat 'kubectl get nodes -o wide'
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh """
                        sed -i 's|your-registry/mern-backend:latest|${DOCKER_REGISTRY}/mern-backend:${IMAGE_TAG}|g' k8s/backend-deployment.yaml
                        sed -i 's|your-registry/mern-frontend:latest|${DOCKER_REGISTRY}/mern-frontend:${IMAGE_TAG}|g' k8s/frontend-deployment.yaml
                    """
                    bat 'kubectl apply -f k8s/'
                    bat 'kubectl rollout status deployment/backend-deployment'
                    bat 'kubectl rollout status deployment/frontend-deployment'
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    bat 'kubectl get pods -o wide'
                    bat 'kubectl get services'
                    bat 'kubectl wait --for=condition=ready pod -l app=backend --timeout=300s'
                    bat 'kubectl wait --for=condition=ready pod -l app=frontend --timeout=300s'
                }
            }
        }
    }

    post {
        success { echo '🎉 Pipeline completed successfully!' }
        failure { echo '❌ Pipeline failed!' }
        always { sh 'docker system prune -f' }
    }
}
