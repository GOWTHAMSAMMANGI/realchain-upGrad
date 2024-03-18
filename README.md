# Node.js REST API with Docker, Kubernetes, Prometheus, and Grafana

This project demonstrates how to build and deploy a Node.js-based RESTful API using Docker and Kubernetes. Additionally, it showcases how to monitor and alert using Prometheus and Grafana.

## Prerequisites

Before you get started, ensure you have the following installed:

- Node.js and npm
- Docker
- HELM
- Kubernetes cluster (e.g., AWS EKS)
- Kubectl
- Prometheus and Grafana set up in your Kubernetes cluster

## Getting Started

Run the following command to start the project:

```bash
mkdir project-2
cd project-2
npm init -y
```

This command will initialize a node.js project in your directory and you have `package.json` file.

### Installing dependencies

For this project we need to install dependencies like `express.js` and `prom-client`. To install the dependencies run the following command:

```bash
npm i express prom-client
```

### Creating configuration file and testing configuration locally

We need a configuration file and we need to create `index.js` file and write the configurations and Run the `node index.js` to test the application whether it is working properly or not.  
Go the the browser and search this address `http://ec2-3-91-182-117.compute-1.amazonaws.com:3000/metrics` and you will see something like this:

![Alt text](image.png)

For more results check the `Screenshots Directory`.

### Containerizing the application

Create a Dockerfile and put the following configuration:

```dockerfile
# Use an official Node.js runtime as the base image
FROM node:14

# Set the working directory in the container
WORKDIR /app

# Copy package.json and package-lock.json to the container
COPY package*.json ./

# Install Node.js dependencies
RUN npm install

# Copy the rest of your application code to the container
COPY . .

# Expose the port your application is running on
EXPOSE 3000

# Define the command to start your Node.js application
CMD [ "node", "index.js" ]
```

## Implement Jenkins CI/CD Pipeline

- Log in to Jenkins server and install some plugins such as:  
a) Docker Pipeline  
b) Pipeline Utilities

- Now click `manage jenkins` and then go to `credentials` and create a new credentials for your dockerhub login. 

- Now create a pipeline job and write the pipeline script. Your Jenkinsfile should look something like this:
```
pipeline {
    agent any
    environment {
        registry = "gowthamsammangi/upgrad-nodejs2"
        registryCredential = 'dockerhub'
    }

    stages {
        stage('git checkout') {
            steps {
                git branch: 'project-2', url: 'https://github.com/GOWTHAMSAMMANGI/Upgrad-Capstone-2-Nodejs.git'
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

...
![Alt text](image-1.png)

........
Step 6:Testing and Verification
Here we will verify that our application is successfully deployed on kubernetes or not.

Now go to the kubernetes cluster and run kubectl get pods you will get something like this:
![Alt text](image-2.png)

### Setting up the Prometheus and Grafana by adding a stage in Jenkins file

To install prometheus and Grafana you need to run the following command

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install my-kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 51.4.0
```
![Alt text](image-4.png)
We are using kube-prometheus-stack because it comes with prometheus all the dependencies which we need to install.

Now edit the service in using `kubectl edit svc` command and expose the grafana service.

Login to Grafana and go to Home > Dashboards > Kubernetes/ComputeResources/Pod and select the pods which we have deployed it looks like this:
![Alt text](image-5.png)