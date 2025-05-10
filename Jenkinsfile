pipeline {
    agent any
    environment{
        DOCKER_IMAGE = "vishalmahawar5200/10may2025"
    }

    stages{
        stage('Install Docker Dependencies'){
            steps{
                sh '''
                    apt update
                    apt upgrade -y
                    apt install -y docker.io
                '''
            }
        }
        
        stage('Start Docker Daemon (if not running)') {
            steps {
                sh '''
                    if ! pgrep dockerd > /dev/null; then
                        echo "Starting Docker Daemon"
                        nohup dockerd > /tmp/dockerd.log 2>&1 &
                        sleep 10
                    else
                        echo "Docker daemon is already running"
                    fi
                '''
            }
        }

        stage('Check Version'){
            steps{
                sh "docker --version"
            }
        }

        stage('Build Image'){
            steps{
                withCredentials([usernamePassword(credentialsId: 'docker-hub-repo', passwordVariable: 'PASS', userVariable: 'USER')]) {
                    sh 'docker build -t ${DOCKER_IMAGE}:latest .'
                    sh "echo $PASS | docker login -u $USER --password-stdin"
                }
            }
        }
    }
}