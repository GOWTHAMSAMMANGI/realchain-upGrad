pipeline {
    agent any
    environment {
        registry = "gowthamsammangi/realchain-upgrad"
        registryCredential = 'dockerhub'
        SCANNER_HOME=tool 'sonar-scanner'
    }

    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('git checkout') {
            steps {
                git branch: 'project-2', url: 'https://github.com/GOWTHAMSAMMANGI/realchain-upGrad.git'
            }
        }

        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=realchain \
                    -Dsonar.projectKey=realchain '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        
        stage('Build docker image') {
            steps {
                script {
                  dockerImage = docker.build registry + ":v$BUILD_NUMBER"
              }
            }
        }
        stage('Upload Image'){
          steps{
            script {
              docker.withRegistry('', registryCredential) {
                dockerImage.push("v$BUILD_NUMBER")
                dockerImage.push('latest')
              }
            }
          }
        }
        stage('Remove Unused docker image') {
          steps{
            sh "docker rmi $registry:v$BUILD_NUMBER"
          }
        }
        stage('Deploying container to Kubernetes') {
           steps {
               script {
                    def serviceExists = ""
                   serviceExists = sh(script: "kubectl get services crud-svc -n default | grep crud-svc | awk '{ print \$1}'", returnStdout: true).trim()
                    echo serviceExists 
                    if (serviceExists == "crud-svc" ) {
                         sh "echo 'Upgrading...'"
                        sh "helm upgrade crudapp crud --set appimage=${registry}:v${BUILD_NUMBER}"
                    
                    } else {
                         sh "echo 'Installing...'"
                         sh "helm install crudapp crud --set appimage=${registry}:v${BUILD_NUMBER}"
                        }
               }
            }
        }    
        stage('Monitoring with Prometheus & Grafana') {
           steps {
                sh "helm repo add prometheus-community https://prometheus-community.github.io/helm-charts"
                sh "helm repo update"
                sh "helm install my-kube-prometheus-stack prometheus-community/kube-prometheus-stack"
            }
        }
        

    }
}
