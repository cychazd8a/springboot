pipeline {
    agent any
    tools {
        maven 'maven'
    }
    environment {
        IMAGE_NAME = 'springbootapp'
        IMAGE_TAG = 'latest'
        TENANT_ID ='a8a56f91-3372-425b-b231-74962efba888'
        ACR_NAME = 'luckyregistryy'
        ACR_LOGIN_SERVER = 'luckyregistryy.azurecr.io'
        FULL_IMAGE_NAME = "${ACR_LOGIN_SERVER}/${IMAGE_NAME}:${IMAGE_TAG}"
        RG              = "demo11"
        NAME            = "lucky-aks-cluster11"
    }
    stages {
        stage('Checkout FROM GIT') {
            steps {
                git branch: 'prod' , url: 'https://github.com/cychazd8a/springboot.git'
        }
      }
        stage('Validate with Maven ') {
            steps {
                sh 'mvn validate'
            }
        }
        stage('Compile with Maven ') {
            steps {
                sh 'mvn compile'
            }
        }
        stage('Sonar Analysis ') {
            environment {
                SCANNER_HOME = tool 'SonarQubeScanner'
            }   
            steps {
                withSonarQubeEnv('SonarQube') {
                    withCredentials([string(credentialsId: 'sonarcloud-token', variable: 'SONAR_TOKEN')]) {
            sh """
                ${SCANNER_HOME}/bin/sonar-scanner \
                -Dsonar.organization=cychazd8a \
                -Dsonar.projectName=springboot \
                -Dsonar.projectKey=cychazd8a_springboot \
                -Dsonar.host.url=https://sonarcloud.io \
                -Dsonar.token=${SONAR_TOKEN} \
                -Dsonar.sources=src \
                -Dsonar.java.binaries=target/classes
            """
                   }
                }
            }         
        }
         stage('Maven Package ') {
            steps {
                sh 'mvn package'
            }
        }
        stage('Sonar Quality Gate') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true, credentialsId: 'sonarserver'
                }
            }
        }
        stage('Docker Build') {
            steps {
                script {
                    echo "Building Docker Image......."
                    docker.build ("${IMAGE_NAME}:${IMAGE_TAG}") 
                }
            }
        }
        stage('Azure Login TO ACR') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'azure-acr-spn', usernameVariable: 'AZURE_USERNAME', passwordVariable: 'AZURE_PASSWORD')]) {
                    script {
                        echo "Azure Login Started"
                        sh '''
                        az login --service-principal -u $AZURE_USERNAME -p $AZURE_PASSWORD --tenant $TENANT_ID
                        az acr login --name $ACR_NAME
                        '''
                    }
                }
            }
        }
        stage('Docker Push to ACR') {
            steps {
                script {
                    echo "Docker Image Push to ACR"
                    sh '''
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${FULL_IMAGE_NAME}
                   
                    docker push ${FULL_IMAGE_NAME}
                    '''
                }
            }
        }
        stage('Azure Login TO AKS') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'azure-acr-spn', usernameVariable: 'AZURE_USERNAME', passwordVariable: 'AZURE_PASSWORD')]) {
                    script {
                        echo "Azure Login to AKS"
                        sh '''
                        az login --service-principal -u $AZURE_USERNAME -p $AZURE_PASSWORD --tenant $TENANT_ID
                        az aks get-credentials --resource-group $RG --name $NAME --overwrite-existing
                        '''
                    }
                }
            }
        }
        stage('Deploy to AKS') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'azure-acr-spn', usernameVariable: 'AZURE_USERNAME', passwordVariable: 'AZURE_PASSWORD')]) {
                    script {
                        echo "Azure Login to AKS"
                        sh '''
                        az login --service-principal -u $AZURE_USERNAME -p $AZURE_PASSWORD --tenant $TENANT_ID
                        kubectl apply -f k8s/sprinboot-deployment.yaml
                        '''
                    }
                }
            }
        }
    }
}