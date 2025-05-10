pipeline {
    agent any
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
        
        stage('Check Version'){
            steps{
                sh "docker --version"
            }
        }
    }
}