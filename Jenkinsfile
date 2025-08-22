pipeline {
    agent any
    environment {
        APP_NAME = "getting-started"
        DOCKER_REGISTRY_USER = "raphaelcarvalho30"
        APP_VERSION = "1.0.0-${env.BUILD_NUMBER}"
        DOCKER_IMAGE_TAGGED = "${DOCKER_REGISTRY_USER}/${APP_NAME}:${APP_VERSION}"
    }

    stages {
        stage('Checkout') {
            steps {
                // Passo 1: Limpa o workspace de builds anteriores
                cleanWs()
                // Passo 2: Baixa o código do repositório
                checkout scm
                echo "Código baixado com sucesso."
            }
        }
        
        stage('Build e Testes') {
            steps {
                echo 'Iniciando a etapa de Build e Testes...'
                // Agora o arquivo mvnw existe e o comando vai funcionar
                sh './mvnw clean package'
            }
        }
        
        stage('Construir e Publicar Imagem no Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        echo "Construindo a imagem Docker: ${DOCKER_IMAGE_TAGGED}"
                        sh "docker build -t ${DOCKER_IMAGE_TAGGED} ."

                        echo "Fazendo login no Docker Hub..."
                        sh "echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin"

                        echo "Publicando a imagem no Docker Hub..."
                        sh "docker push ${DOCKER_IMAGE_TAGGED}"
                        
                        echo "Imagem ${DOCKER_IMAGE_TAGGED} publicada com sucesso."
                    }
                }
            }
        }

        stage('Deploy em DEV') {
            //when {
                // Usando a variável de ambiente para ser mais explícito
                //expression { env.BRANCH_NAME == 'dev' }
            //}
            steps {
                script {
                    def targetNamespace = 'des'
                    def deploymentName = "${APP_NAME}-des"
                    def deploymentFile = "./kubernetes/${targetNamespace}/deployment.yaml"
                    def updatedDeploymentFile = "./kubernetes/${targetNamespace}/deployment-updated.yaml"
                    
                    echo "Iniciando deploy para o ambiente de Desenvolvimento (Branch: ${env.BRANCH_NAME})"
                    sh "minikube kubectl -- create namespace ${targetNamespace} || true"
                    
                    echo "Atualizando o deployment para usar a imagem ${DOCKER_IMAGE_TAGGED}"
                    sh "sed 's|image: .*|image: ${DOCKER_IMAGE_TAGGED}|g' ${deploymentFile} > ${updatedDeploymentFile}"
                    
                    sh "minikube kubectl -- apply -f ${updatedDeploymentFile}"
                    sh "minikube kubectl -- apply -f ./kubernetes/${targetNamespace}/service.yaml"
                    
                    echo "Aguardando o Deployment ser concluido..."
                    sh "minikube kubectl -- rollout status deployment/${deploymentName} -n ${targetNamespace}"
                    
                    echo "Deploy para DES concluido com sucesso."
                }
            }
        }

        stage('Deploy em PRD') {
            // when {
                // Usando a variável de ambiente para ser mais explícito
                //expression { env.BRANCH_NAME == 'main' }
            //}
            steps {
                script {
                    def targetNamespace = 'prd'
                    def deploymentName = "${APP_NAME}-prd"
                    def deploymentFile = "./kubernetes/${targetNamespace}/deployment.yaml"
                    def updatedDeploymentFile = "./kubernetes/${targetNamespace}/deployment-updated.yaml"
                    
                    echo "Iniciando deploy para o ambiente de Produção (Branch: ${env.BRANCH_NAME})"
                    sh "minikube kubectl -- create namespace ${targetNamespace} || true"
                    
                    echo "Atualizando o deployment para usar a imagem ${DOCKER_IMAGE_TAGGED}"
                    sh "sed 's|image: .*|image: ${DOCKER_IMAGE_TAGGED}|g' ${deploymentFile} > ${updatedDeploymentFile}"

                    sh "minikube kubectl -- apply -f ${updatedDeploymentFile}"
                    sh "minikube kubectl -- apply -f ./kubernetes/${targetNamespace}/service.yaml"
                    
                    echo "Aguardando o Deployment ser concluido..."
                    sh "minikube kubectl -- rollout status deployment/${deploymentName} -n ${targetNamespace}"
                    
                    echo "Deploy para PRD concluido com sucesso."
                }
            }
        }
    }
    post {
        always {
            sh "docker logout"
            sh "rm -f ./kubernetes/des/deployment-updated.yaml ./kubernetes/prd/deployment-updated.yaml"
        } // AAAAAHHHHHHHHHHH
    }
}