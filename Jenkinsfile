pipeline {
    agent any

    tools {
        jdk 'jdk17' // Java version if required for other tasks
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner' // SonarQube scanner tool configuration
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/wissem-saidi/simple-project.git'
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
                    sh '''$SCANNER_HOME/bin/sonar-scanner \\
                            -Dsonar.projectKey=webapp \\
                            -Dsonar.projectName=webapp \\
                            -Dsonar.projectVersion=1.0 \\
                            -Dsonar.sources=.'''
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-cred') {
                        sh "docker build -t saidiwissem/web-app:latest ."
                        // Optionally push the image to Docker Hub
                        sh "docker push saidiwissem/web-app:latest"
                    }
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html saidiwissem/web-app:latest"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'kubeconfig', serverUrl: 'https://192.168.1.210:6443') {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeConfig(credentialsId: 'kubeconfig', serverUrl: 'https://192.168.1.210:6443') {
                    script {
                        def podsStatus = sh(script: "kubectl get pods -n default", returnStdout: true).trim()
                        def svcStatus = sh(script: "kubectl get svc -n default", returnStdout: true).trim()
                        echo "Pods Status:\n${podsStatus}"
                        echo "Services Status:\n${svcStatus}"
                        if (podsStatus.contains('0/1') || svcStatus.contains('0')) {
                            error "Failed to verify Kubernetes deployment"
                        }
                    }
                }
            }
        }

        stage('Output Service URL') {
            steps {
                script {
                    def serviceUrl = "http://192.168.1.211:30007"
                    echo "Access your application at: ${serviceUrl}"
                }
            }
        }
    }

    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

                def body = """
                    <html>
                    <body>
                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding: 10px;">
                    <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                    </div>
                    </body>
                    </html>
                """

                emailext (
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'scrpngldn6@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }
    }
}
