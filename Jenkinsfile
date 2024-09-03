pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    
    environment {
        DOCKER_IMAGE = "saidiwissem/webapp:latest"
        KUBECONFIG_CREDENTIALS = credentials('k8s') // Kubernetes credentials in Jenkins
        SCANNER_HOME= tool 'sonar-scanner'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', credentialsId: 'gitcred', url: 'https://github.com/wissem-saidi/simple-project.git'
            }
        }
        
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=webapp  -Dsonar.projectKey=webapp  \\
                            -Dsonar.sources=. '''
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script {
                  waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            }
        }
        
        stage('Publish To Nexus') {
            steps {
               withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
               script {
                   withDockerRegistry(credentialsId: 'dockercred', toolName: 'docker') {
                        sh "docker build -t saidiwissem/webapp:latest ."
                    }
               }
            }
        }
        
        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html saidiwissem/webapp:latest "
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockercred', toolName: 'docker') {
                        sh "docker push saidiwissem/webapp:latest"
                    }
                }
            }
        }

        // New stage for deploying to Minikube
        stage('Deploy to Minikube') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'k8s']) {
                        sh "kubectl apply -f deployment-service.yaml"
                    }
                }
            }
        }
    }
}
