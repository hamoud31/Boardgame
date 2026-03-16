pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/hamoud31/Boardgame.git'
            }
        }

        stage('Gitleaks') {
            steps {
                sh 'gitleaks detect --source . -r gitleaks-report.json --redact'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --format template --template "@contrib/html.tpl" -o trivy-report.html . || true'
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonnar') {
                    sh """
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=boardgame \
                    -Dsonar.projectName=Boardgame \
                    -Dsonar.sources=. \
                    -Dsonar.java.binaries=target/classes
                    """
                }
            }
        }

        stage('QualityGate Check') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn package'
            }
        }

        stage('Deploy to Maven Repo') {
            steps {
                withMaven(globalMavenSettingsConfig: 'gmaven', jdk: 'jdk17', maven: 'maven3', traceability: true) {
                    sh 'mvn deploy'
                }
            }
        }

        stage('Docker Build & Tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh 'docker build -t hamoud07/boardgame:latest .'
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image --format json -o trivy-image-report.json hamoud07/boardgame:latest'
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh 'docker push hamoud07/boardgame:latest'
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8s-token', namespace: 'jenkins-deploy', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.2.52:6443') {
                        sh 'kubectl apply -f deployment-service.yaml'
                        sh 'kubectl get pods,svc  -n jenkins-deploy '
                    }
                }
            }
        }

    } 
} 

