pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node18'
    }
    environment {
        SCANNER_HOME = tool name: 'sonar-scanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
    }
    stages {
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/Sunilmargale/Uptime-kuma-CI-CD.git'
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=uptime \
                    -Dsonar.projectKey=uptime
                    '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    def qualityGate = waitForQualityGate()
                    if (qualityGate.status != 'OK') {
                        error "Pipeline aborted due to quality gate failure: ${qualityGate.status}"
                    }
                }
            }
        }
        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DC'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Trivy File System Scan') {
            steps {
                sh "trivy fs . > trivyfs.json"
            }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t uptime ."
                        sh "docker tag uptime sunilmargale/uptime:latest"
                        sh "docker push sunilmargale/uptime:latest"
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh "trivy image sunilmargale/uptime:latest > trivy.json"
            }
        }
        stage('Remove Existing Container') {
            steps {
                sh "docker stop uptime || true"
                sh "docker rm uptime || true"
            }
        }
        stage('Deploy to Container') {
            steps {
                sh 'docker run -d --name uptime -v /var/run/docker.sock:/var/run/docker.sock -p 3001:3001 sunilmargale/uptime:latest'
            }
        }
    }
}
