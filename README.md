# 🎮 DevSecOps: Deploying the 2048 Game on Docker and Kubernetes with Jenkins CI/CD

![DevSecOps Banner](https://miro.medium.com/v2/resize:fit:1200/1*eNpn8LoI8KJ9az3dbmqltA.png)

---

## 📘 Project Overview

This project demonstrates a **complete DevSecOps pipeline** for the **2048 Game** — a popular puzzle web application.  
It integrates **Jenkins CI/CD**, **SonarQube**, **OWASP Dependency-Check**, **Trivy**, **Docker**, **Kubernetes**, and **Prometheus–Grafana** monitoring, all deployed on **AWS EC2 instances**.

The pipeline automatically:
- Analyzes code quality  
- Detects vulnerabilities  
- Builds and scans Docker images  
- Deploys securely to Kubernetes  
- Monitors infrastructure and pods using Prometheus and Grafana  

---

## 🧩 Tech Stack

| Category | Tools & Technologies |
|-----------|----------------------|
| **Version Control** | Git & GitHub |
| **CI/CD Orchestration** | Jenkins |
| **Code Quality Analysis** | SonarQube |
| **Dependency Security** | OWASP Dependency-Check |
| **Container Security** | Trivy |
| **Containerization** | Docker |
| **Orchestration & Deployment** | Kubernetes (kubeadm setup) |
| **Monitoring & Visualization** | Prometheus + Grafana |
| **Cloud Infrastructure** | AWS EC2 (Ubuntu 24.04 LTS) |
| **Programming Language** | Node.js / Python |
| **Web Server** | NGINX |

---

## ⚙️ Pipeline Workflow

1. **Clean Workspace** → Clears previous build files.  
2. **Checkout Code** → Pulls the latest code from GitHub.  
3. **SonarQube Analysis** → Scans for code smells, bugs, and vulnerabilities.  
4. **Install Dependencies** → Installs NPM dependencies for the 2048 app.  
5. **OWASP Dependency Check** → Detects known vulnerable libraries.  
6. **Trivy Filesystem Scan** → Scans the local filesystem for HIGH/CRITICAL CVEs.  
7. **Docker Build & Push** → Builds image and pushes it to Docker Hub.  
8. **Trivy Image Scan** → Scans the built Docker image.  
9. **Deploy to Container** → Runs the app locally in a Docker container.  
10. **Deploy to Kubernetes** → Deploys the containerized app to K8s using YAML manifests.  

---

## 📜 Complete Jenkins Declarative Pipeline

```groovy
pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'nodejs24'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git 'https://github.com/Shuvro-373/2048_game.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=Game \
                        -Dsonar.projectName=Game'''
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('OWASP Dependency-Check') {
            steps {
                dependencyCheck additionalArguments: '--scan . --format XML', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                archiveArtifacts artifacts: '**/dependency-check-report.*', onlyIfSuccessful: true
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                    trivy fs . \
                    --format table \
                    --severity HIGH,CRITICAL \
                    --no-progress > trivyfs.txt
                '''
                archiveArtifacts artifacts: 'trivyfs.txt', onlyIfSuccessful: true
            }
        }

        stage("Docker Build & Push") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub-creds') {
                        sh '''
                            docker build -t 2048 .
                            docker tag 2048 shuvro373/2048:latest
                            docker push shuvro373/2048:latest
                        '''
                    }
                }
            }
        }

        stage("Trivy Image Scan") {
            steps {
                sh '''
                    trivy image shuvro373/2048:latest \
                    --severity HIGH,CRITICAL \
                    --format table \
                    --no-progress > trivy.txt
                '''
                archiveArtifacts artifacts: 'trivy.txt', onlyIfSuccessful: true
            }
        }

        stage('Deploy to container') {
            steps {
                sh '''
                    docker rm -f 2048 || true
                    docker run -d --name 2048 -p 3000:3000 shuvro373/2048:latest
                '''
            }
        }

        stage('Deploy to kubernets'){
            steps{
                script{
                    withKubeConfig(
                        caCertificate: '', 
                        clusterName: '', 
                        contextName: '', 
                        credentialsId: 'k8s', 
                        namespace: '', 
                        restrictKubeConfigAccess: false, 
                        serverUrl: ''
                    ) {
                        sh 'kubectl apply -f deployment.yaml'
                    }
                }
            }
        }
    }
}
