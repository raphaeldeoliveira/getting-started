pipeline {
    agent any
    environment {
        APP_NAME = "getting-started"
        DOCKER_REGISTRY_USER = "raphaelcarvalho30"
        APP_VERSION = "1.0.0-${env.BUILD_NUMBER}"
        DOCKER_PUSH_URL = "${DOCKER_REGISTRY_USER}/${APP_NAME}:${APP_VERSION}"
        DOCKER_IMAGE_LATEST = "${DOCKER_REGISTRY_USER}/${APP_NAME}:latest"
    }

    stages {
        stage('Build e Testes') {
            steps {
                echo 'Iniciando a etapa de Build e Testes...'
                // Garante que o workspace seja limpo e o repositorio clonado
                cleanWs()
                checkout scm

                // Passo 1: Compilar a aplicação
                sh './mvnw clean package'
                
                // Passo 2: Executar testes unitários
                sh './mvnw test'
            }
        }
        
        stage('Construir e Publicar Imagem') {
            steps {
                script {
                    // A tag é criada com o numero da build
                    echo "Passo 3: Construindo e publicando a imagem Docker..."
                    sh "docker build -t ${DOCKER_PUSH_URL} ."
                    sh "docker push ${DOCKER_PUSH_URL}"
                    sh "docker tag ${DOCKER_PUSH_URL} ${DOCKER_IMAGE_LATEST}"
                    sh "docker push ${DOCKER_IMAGE_LATEST}"
                    
                    echo "Imagem ${APP_VERSION} publicada com sucesso no Docker Hub."
                }
            }
        }

        stage('Deploy em DEV') {
            when {
                branch 'dev' // Este estágio só roda se a branch for 'dev'
            }
            steps {
                script {
                    def targetNamespace = 'des'
                    
                    echo "Passo 4: Iniciando deploy para o ambiente de Desenvolvimento..."
                    
                    sh "minikube kubectl -- create namespace ${targetNamespace} || true"
                    sh "minikube kubectl -- apply -f ./kubernetes/${targetNamespace}/deployment.yaml"
                    sh "minikube kubectl -- apply -f ./kubernetes/${targetNamespace}/service.yaml"
                    
                    echo "Aguardando o Deployment ser concluido..."
                    sh "minikube kubectl -- rollout status deployment/${APP_NAME} -n ${targetNamespace}"
                    
                    echo "Passo 5: Validando a aplicacao..."
                    def serviceUrl = sh(script: "minikube service ${APP_NAME}-service-${targetNamespace} --namespace=${targetNamespace} --url", returnStdout: true).trim()
                    sh "curl --fail --silent ${serviceUrl}/hello"
                    
                    echo "Deploy e validacao para DES concluido com sucesso."
                }
            }
        }

        stage('Deploy em PRD') {
            when {
                branch 'main' // Este estágio só roda se a branch for 'main'
            }
            steps {
                script {
                    def targetNamespace = 'prd'
                    
                    echo "Passo 4: Iniciando deploy para o ambiente de Producao..."
                    
                    sh "minikube kubectl -- create namespace ${targetNamespace} || true"
                    sh "minikube kubectl -- apply -f ./kubernetes/${targetNamespace}/deployment.yaml"
                    sh "minikube kubectl -- apply -f ./kubernetes/${targetNamespace}/service.yaml"
                    
                    echo "Aguardando o Deployment ser concluido..."
                    sh "minikube kubectl -- rollout status deployment/${APP_NAME} -n ${targetNamespace}"
                    
                    echo "Passo 5: Validando a aplicacao..."
                    def serviceUrl = sh(script: "minikube service ${APP_NAME}-service-${targetNamespace} --namespace=${targetNamespace} --url", returnStdout: true).trim()
                    sh "curl --fail --silent ${serviceUrl}/hello"
                    
                    echo "Deploy e validacao para PRD concluido com sucesso."
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline executada com Sucesso"
        }
        failure {
            echo "Pipeline executada com Falha"
        }
    }
}