pipeline {
    agent any
    environment {
        APP_NAME = "getting-started"
        DOCKER_REGISTRY_USER = "raphaelcarvalho30"
        APP_VERSION = "1.0.0-${env.BUILD_NUMBER}"
        DOCKER_IMAGE_TAGGED = "${DOCKER_REGISTRY_USER}/${APP_NAME}:${APP_VERSION}"
    }

    stages {
        stage('Build e Testes') {
            steps {
                echo 'Iniciando a etapa de Build e Testes...'
                cleanWs()
                sh './mvnw clean package'
            }
        }
        
        stage('Construir e Publicar Imagem no Docker Hub') {
            steps {
                // Aqui você precisará configurar as credenciais do Docker Hub no Jenkins
                // e envolvê-las no comando de push.
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
            when {
                branch 'dev'
            }
            steps {
                script {
                    def targetNamespace = 'des'
                    // IMPORTANTE: O nome do deployment no seu YAML é 'getting-started-des'
                    def deploymentName = "${APP_NAME}-des"
                    def deploymentFile = "./kubernetes/${targetNamespace}/deployment.yaml"
                    def updatedDeploymentFile = "./kubernetes/${targetNamespace}/deployment-updated.yaml"
                    
                    echo "Iniciando deploy para o ambiente de Desenvolvimento..."
                    sh "minikube kubectl -- create namespace ${targetNamespace} || true"
                    
                    // ATUALIZA O ARQUIVO DE DEPLOYMENT COM A NOVA IMAGEM
                    echo "Atualizando o deployment para usar a imagem ${DOCKER_IMAGE_TAGGED}"
                    sh "sed 's|image: .*|image: ${DOCKER_IMAGE_TAGGED}|g' ${deploymentFile} > ${updatedDeploymentFile}"
                    
                    // APLICA O ARQUIVO ATUALIZADO
                    sh "minikube kubectl -- apply -f ${updatedDeploymentFile}"
                    sh "minikube kubectl -- apply -f ./kubernetes/${targetNamespace}/service.yaml"
                    
                    echo "Aguardando o Deployment ser concluido..."
                    sh "minikube kubectl -- rollout status deployment/${deploymentName} -n ${targetNamespace}"
                    
                    echo "Deploy para DES concluido com sucesso."
                }
            }
        }

        // Você pode criar um estágio 'Deploy em PRD' similar, se necessário.
    }
    post {
        always {
            // Logout do Docker Hub e limpeza
            sh "docker logout"
            sh "rm -f ./kubernetes/des/deployment-updated.yaml ./kubernetes/prd/deployment-updated.yaml"
        }
    }
}