pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'ajiththomas10'
        IMAGE_TAG = "${BUILD_NUMBER}"
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
                }
            }
        }

        stage('Install Dependencies & Test Frontend') {
            steps {
                dir('frontend') {
                    bat 'npm install'
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

        stage('Check kubectl availability') {
            steps {
                bat 'where kubectl'
                bat 'kubectl version --client'
            }
        }

        stage('Verify Kubernetes Connectivity') {
            steps {
                echo '🔍 Verifying Kubernetes API (kubeconfig-based access)...'
                bat 'kubectl cluster-info'
                bat 'kubectl get nodes -o wide'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                bat """
                powershell -Command "(Get-Content k8s/backend-deployment.yaml) -replace 'your-registry/mern-backend:latest','${DOCKER_REGISTRY}/mern-backend:${IMAGE_TAG}' | Set-Content k8s/backend-deployment.yaml"
                powershell -Command "(Get-Content k8s/frontend-deployment.yaml) -replace 'your-registry/mern-frontend:latest','${DOCKER_REGISTRY}/mern-frontend:${IMAGE_TAG}' | Set-Content k8s/frontend-deployment.yaml"
                """

                bat 'kubectl apply -f k8s/'
                bat 'kubectl rollout status deployment/backend-deployment'
                bat 'kubectl rollout status deployment/frontend-deployment'
            }
        }

        stage('Verify Deployment') {
            steps {
                bat 'kubectl get pods -o wide'
                bat 'kubectl get services'
                bat 'kubectl wait --for=condition=ready pod -l app=backend --timeout=300s'
                bat 'kubectl wait --for=condition=ready pod -l app=frontend --timeout=300s'
            }
        }
    }

    post {
        success { echo '🎉 Pipeline completed successfully!' }
        failure { echo '❌ Pipeline failed!' }
        always { bat 'docker system prune -f' }
    }
}
