pipeline {
    agent any

    environment {
        // Mapeando as branches para os ambientes
        IMAGE_NAME = "getting-started"
        IMAGE_TAG = "1.0.0-${BUILD_NUMBER}" // Usando o numero da build como tag
        DOCKER_REGISTRY_USER = "raphaelcarvalho30"
    }

    stages {
        stage('Build e Testes Condicionais') {
            steps {
                script {
                    //def branch = env.BRANCH_NAME
                    def branch = des
                    if (branch == 'dev' || branch == 'main') {
                        echo "Iniciando build e testes para a branch ${branch}"
                        
                        // Passo 1: Compilar a aplicação
                        sh './mvnw clean package'

                        // Passo 2: Executar testes unitários
                        sh './mvnw test'
                    } else {
                        echo "Branch ${branch} nao tem pipeline de build configurada. Ignorando."
                    }
                }
            }
        }
        
        stage('Construir e Publicar a Imagem') {
            steps {
                script {
                    def branch = env.BRANCH_NAME
                    if (branch == 'dev' || branch == 'main') {
                        echo "Passo 3: Construindo e publicando a imagem Docker..."
                        def imageTag = "${DOCKER_REGISTRY_USER}/${IMAGE_NAME}:${env.APP_VERSION}"
                        
                        // Constroi a imagem com a tag correta
                        sh "docker build -t ${imageTag} ."
                        
                        // Faz o push para o Docker Hub
                        sh "docker push ${imageTag}"
                        
                        echo "Imagem ${imageTag} publicada com sucesso no Docker Hub."
                    }
                }
            }
        }

        stage('Deploy Condicional') {
            steps {
                script {
                    def branch = env.BRANCH_NAME
                    def targetNamespace = ''
                    if (branch == 'dev') {
                        targetNamespace = 'des'
                    } else if (branch == 'main') {
                        targetNamespace = 'prd'
                    }
                    
                    if (targetNamespace != '') {
                        echo "Passo 4: Iniciando deploy para o ambiente ${targetNamespace}..."
                        
                        // Aplica os manifestos do Kubernetes
                        sh "minikube kubectl -- create namespace ${targetNamespace} || true"
                        sh "minikube kubectl -- apply -f kubernetes/${targetNamespace}/deployment.yaml"
                        sh "minikube kubectl -- apply -f kubernetes/${targetNamespace}/service.yaml"
                        
                        echo "Aguardando o Deployment ser concluido..."
                        sh "minikube kubectl -- rollout status deployment/${IMAGE_NAME}-${targetNamespace} -n ${targetNamespace}"
                        
                        echo "Passo 5: Validando a aplicacao no ambiente ${targetNamespace}..."
                        def serviceUrl = sh(script: "minikube service ${IMAGE_NAME}-service-${targetNamespace} --namespace=${targetNamespace} --url", returnStdout: true).trim()
                        sh "curl --fail --silent ${serviceUrl}/hello"
                        
                        echo "Deploy e validacao para ${targetNamespace} concluido com sucesso."
                    }
                }
            }
        }
    }
// aaa
    post {
        success {
            echo "Pipeline executada com Sucesso"
        }
        failure {
            echo "Pipeline executada com Falha"
        }
    }
}