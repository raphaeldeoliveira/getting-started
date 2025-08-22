pipeline {
    agent any
    environment {
        APP_NAME = "getting-started"
        DOCKER_REGISTRY_USER = "raphaelcarvalho30" // Pode manter para dar nome à imagem
        APP_VERSION = "1.0.0-${env.BUILD_NUMBER}"
        // Renomeei a variável para refletir que é o nome+tag da imagem
        DOCKER_IMAGE_TAGGED = "${DOCKER_REGISTRY_USER}/${APP_NAME}:${APP_VERSION}"
    }

    stages {
        stage('Build e Testes') {
            steps {
                echo 'Iniciando a etapa de Build e Testes...'
                // Garante que o workspace seja limpo antes de começar
                cleanWs()

                // O checkout do código já é feito automaticamente no início do pipeline declarativo

                // Passo 1 e 2: Compilar a aplicação e executar testes
                // O comando 'package' já executa a fase de 'test', então o segundo comando era redundante.
                sh './mvnw clean package'
            }
        }
        
        stage('Construir e Carregar Imagem Local') {
            steps {
                script {
                    echo "Passo 3: Construindo a imagem Docker localmente: ${DOCKER_IMAGE_TAGGED}"
                    sh "docker build -t ${DOCKER_IMAGE_TAGGED} ."

                    echo "Carregando a imagem para dentro do Minikube..."
                    sh "minikube image load ${DOCKER_IMAGE_TAGGED}"
                    
                    echo "Imagem ${DOCKER_IMAGE_TAGGED} carregada com sucesso no Minikube."
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
                    def deploymentFile = "./kubernetes/${targetNamespace}/deployment.yaml"
                    def updatedDeploymentFile = "./kubernetes/${targetNamespace}/deployment-updated.yaml"
                    
                    echo "Passo 4: Iniciando deploy para o ambiente de Desenvolvimento..."
                    
                    // Cria o namespace se ele não existir
                    sh "minikube kubectl -- create namespace ${targetNamespace} || true"
                    
                    // Atualiza dinamicamente o arquivo de deployment com a nova tag da imagem
                    echo "Atualizando o arquivo de deployment para usar a imagem ${DOCKER_IMAGE_TAGGED}"
                    sh "sed 's|image: .*|image: ${DOCKER_IMAGE_TAGGED}|g' ${deploymentFile} > ${updatedDeploymentFile}"
                    
                    // Aplica os arquivos YAML atualizados
                    sh "minikube kubectl -- apply -f ${updatedDeploymentFile}"
                    sh "minikube kubectl -- apply -f ./kubernetes/${targetNamespace}/service.yaml"
                    
                    echo "Aguardando o Deployment ser concluido..."
                    sh "minikube kubectl -- rollout status deployment/${APP_NAME} -n ${targetNamespace}"
                    
                    echo "Passo 5: Validando a aplicacao..."
                    // A validação com minikube service pode ser instável em scripts, mas mantida conforme original
                    def serviceUrl = sh(script: "minikube service ${APP_NAME}-service --namespace=${targetNamespace} --url", returnStdout: true).trim()
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
                    def deploymentFile = "./kubernetes/${targetNamespace}/deployment.yaml"
                    def updatedDeploymentFile = "./kubernetes/${targetNamespace}/deployment-updated.yaml"
                    
                    echo "Passo 4: Iniciando deploy para o ambiente de Producao..."
                    
                    sh "minikube kubectl -- create namespace ${targetNamespace} || true"
                    
                    // Atualiza dinamicamente o arquivo de deployment com a nova tag da imagem
                    echo "Atualizando o arquivo de deployment para usar a imagem ${DOCKER_IMAGE_TAGGED}"
                    sh "sed 's|image: .*|image: ${DOCKER_IMAGE_TAGGED}|g' ${deploymentFile} > ${updatedDeploymentFile}"

                    // Aplica os arquivos YAML atualizados
                    sh "minikube kubectl -- apply -f ${updatedDeploymentFile}"
                    sh "minikube kubectl -- apply -f ./kubernetes/${targetNamespace}/service.yaml"
                    
                    echo "Aguardando o Deployment ser concluido..."
                    sh "minikube kubectl -- rollout status deployment/${APP_NAME}-prd -n ${targetNamespace}" // Nome do deployment em PRD
                    
                    echo "Passo 5: Validando a aplicacao..."
                    def serviceUrl = sh(script: "minikube service ${APP_NAME}-service-prd --namespace=${targetNamespace} --url", returnStdout: true).trim()
                    sh "curl --fail --silent ${serviceUrl}/hello"
                    
                    echo "Deploy e validacao para PRD concluido com sucesso."
                }
            }
        }
    }

    post {
        always {
            // Limpa o arquivo temporário de deployment para evitar lixo no workspace
            sh "rm -f ./kubernetes/des/deployment-updated.yaml ./kubernetes/prd/deployment-updated.yaml"
        }
        success {
            echo "Pipeline executada com Sucesso"
        }
        failure {
            echo "Pipeline executada com Falha"
        }
    }
}