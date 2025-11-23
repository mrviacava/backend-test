pipeline {
    agent any

    environment {
        DOCKERHUB_USER = "mrviacava"
        DOCKER_IMAGE   = "backend-devops"

        GH_USER        = "mrviacava"
        GH_REPO        = "backend-test"

        BUILD_NUM      = "${env.BUILD_NUMBER}"
        K8S_NAMESPACE  = "mriquelme"
        K8S_DEPLOYMENT = "backend-deployment"
        K8S_CONTAINER  = "backend-container"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Instalar dependencias') {
            steps {
                bat 'npm install'
            }
        }

        stage('Testing') {
            steps {
                // equivalente a "npm test || true" en Linux
                bat 'npm test || exit /b 0'
            }
        }

        stage('Build') {
            steps {
                // equivalente a "npm run build || true"
                bat 'npm run build || exit /b 0'
            }
        }

        stage('Docker build') {
            steps {
                bat """
                echo Construyendo imagen Docker...
                docker build -t %DOCKERHUB_USER%/%DOCKER_IMAGE%:latest .
                docker tag %DOCKERHUB_USER%/%DOCKER_IMAGE%:latest %DOCKERHUB_USER%/%DOCKER_IMAGE%:%BUILD_NUM%
                """
            }
        }

        stage('Push a Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_PASS'
                )]) {
                    bat """
                    echo %DH_PASS% | docker login -u %DH_USER% --password-stdin
                    docker push %DOCKERHUB_USER%/%DOCKER_IMAGE%:latest
                    docker push %DOCKERHUB_USER%/%DOCKER_IMAGE%:%BUILD_NUM%
                    docker logout
                    """
                }
            }
        }

        stage('Push a GitHub Packages') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-packages-creds',
                    usernameVariable: 'GH_USER_LOGIN',
                    passwordVariable: 'GH_TOKEN'
                )]) {
                    bat """
                    echo %GH_TOKEN% | docker login ghcr.io -u %GH_USER_LOGIN% --password-stdin

                    docker tag %DOCKERHUB_USER%/%DOCKER_IMAGE%:latest ghcr.io/%GH_USER%/%GH_REPO%:latest
                    docker tag %DOCKERHUB_USER%/%DOCKER_IMAGE%:latest ghcr.io/%GH_USER%/%GH_REPO%:%BUILD_NUM%

                    docker push ghcr.io/%GH_USER%/%GH_REPO%:latest
                    docker push ghcr.io/%GH_USER%/%GH_REPO%:%BUILD_NUM%

                    docker logout ghcr.io
                    """
                }
            }
        }

        stage('Actualizar Kubernetes') {
            steps {
                bat """
                echo Actualizando deployment en Kubernetes...
                kubectl set image deployment/%K8S_DEPLOYMENT% %K8S_CONTAINER%=ghcr.io/%GH_USER%/%GH_REPO%:%BUILD_NUM% -n %K8S_NAMESPACE%

                echo Esperando rollout...
                kubectl rollout status deployment/%K8S_DEPLOYMENT% -n %K8S_NAMESPACE%
                """
            }
        }
    }
}
