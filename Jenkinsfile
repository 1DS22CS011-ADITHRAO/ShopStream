pipeline {
    agent any

    environment {
        DOCKER_HUB_USERNAME         = "casualentity673"
        DOCKER_CREDENTIALS_ID       = "docker-creds"
        KUBECONFIG_CREDENTIALS_ID   = "minikube-config"
        MINIKUBE_IP                 = "192.168.49.2"
        FRONTEND_IMAGE_NAME         = "${DOCKER_HUB_USERNAME}/shopstream-frontend"
        BACKEND_IMAGE_NAME          = "${DOCKER_HUB_USERNAME}/shopstream-backend"
        K8S_NAMESPACE               = "shopstream"
        NEXT_PUBLIC_API_URL_BUILD_ARG = "http://${MINIKUBE_IP}:30002/api"

        SPRING_DATASOURCE_PASSWORD = "spring-datasource-password"
        POSTGRES_PASSWORD = "postgres-password"
        JWT_SECRET = "jwt-secret"
        K8S_SECRET_NAME = "shopstream-secrets"
    }

    stages {

        stage('Build & Push Frontend Image') {
            steps {
                script {
                    def dockerfilePath = 'scripts\\docker\\frontend.Dockerfile'
                    def imageName = "${env.FRONTEND_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    def fullImageNameWithRegistry = "docker.io/${imageName}"

                    docker.build(imageName, "--build-arg NEXT_PUBLIC_API_URL_ARG=${env.NEXT_PUBLIC_API_URL_BUILD_ARG} -f ${dockerfilePath} .")

                    withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDENTIALS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        bat 'echo Attempting to login to Docker Hub...'
                        bat "echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin docker.io"
                        echo "Attempting to push image: ${fullImageNameWithRegistry}"
                        bat "docker push ${fullImageNameWithRegistry}"
                    }
                }
            }
        }

        stage('Build & Push Backend Image') {
            steps {
                script {
                    def dockerfilePath = 'scripts\\docker\\backend.Dockerfile'
                    def imageName = "${env.BACKEND_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    def fullImageNameWithRegistry = "docker.io/${imageName}"

                    docker.build(imageName, "-f ${dockerfilePath} .")

                    withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDENTIALS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        bat 'echo Attempting to login to Docker Hub...'
                        bat "echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin docker.io"
                        echo "Attempting to push image: ${fullImageNameWithRegistry}"
                        bat "docker push ${fullImageNameWithRegistry}"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig([credentialsId: 'minikube-config']) {
                    withCredentials([
                        string(credentialsId: env.SPRING_DATASOURCE_PASSWORD, variable: 'SPRING_DATASOURCE_PASSWORD'),
                        string(credentialsId: env.POSTGRES_PASSWORD, variable: 'POSTGRES_PASSWORD'),
                        string(credentialsId: env.JWT_SECRET, variable: 'JWT_SECRET')
                    ]) {
                        script {
                            bat "kubectl config current-context"
                            bat "kubectl apply -f scripts\\k8s\\namespace.yaml"
                            bat "kubectl apply -f scripts\\k8s\\configmap.yaml"

                            // Windows multi-line command using ^
                            bat """
kubectl create secret generic ${env.K8S_SECRET_NAME} ^
  --from-literal=SPRING_DATASOURCE_PASSWORD=%SPRING_DATASOURCE_PASSWORD% ^
  --from-literal=POSTGRES_PASSWORD=%POSTGRES_PASSWORD% ^
  --from-literal=JWT_SECRET=%JWT_SECRET% ^
  -n ${env.K8S_NAMESPACE} ^
  --dry-run=client -o yaml | kubectl apply -f -
"""
                            bat "kubectl get secret ${env.K8S_SECRET_NAME} -n ${env.K8S_NAMESPACE} -o yaml"
                            bat "kubectl apply -f scripts\\k8s\\postgres-pvc.yaml"
                            bat "kubectl apply -f scripts\\k8s\\services.yaml"
                            bat "kubectl apply -f scripts\\k8s\\postgres-deployment.yaml"

                            def frontendFullImage = "${env.FRONTEND_IMAGE_NAME}:${env.BUILD_NUMBER}"
                            def backendFullImage = "${env.BACKEND_IMAGE_NAME}:${env.BUILD_NUMBER}"

                            bat "kubectl set image deployment/shopstream-frontend frontend=${frontendFullImage} -n ${env.K8S_NAMESPACE}"
                            bat "kubectl set image deployment/shopstream-backend backend=${backendFullImage} -n ${env.K8S_NAMESPACE}"

                            bat "kubectl rollout restart deployment/shopstream-backend -n ${env.K8S_NAMESPACE}"

                            bat "kubectl rollout status deployment/shopstream-frontend -n ${env.K8S_NAMESPACE} --timeout=120s"
                            bat "kubectl rollout status deployment/shopstream-backend -n ${env.K8S_NAMESPACE} --timeout=120s"
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished.'
            // cleanWs()
        }
    }
}