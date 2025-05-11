pipeline {
    agent any
    environment {
        DOCKER_IMAGE = "vishalmahawar5200/10may2025"
        DEPLOY_USER = "root"
        DEPLOY_HOST = "65.108.149.166"
    }

    stages {
        stage('Install Docker dependencies') {
            steps {
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

        stage('Check Version') {
            steps {
                sh "docker --version"
            }
        }

        stage('Build Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                    sh 'docker build -t ${DOCKER_IMAGE}:t1 .'
                    sh "echo $PASS | docker login -u $USER --password-stdin"
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        def imageTag = "v${env.BUILD_NUMBER}"
                        sh "docker tag ${DOCKER_IMAGE}:t1 ${DOCKER_IMAGE}:${imageTag}"
                        sh "docker push ${DOCKER_IMAGE}:${imageTag}"
                    }
                }
            }
        }

        //Stage is a function
        stage('check'){
            steps{
                script{
                    //def is keyword to define variable
                    def buildNumberStr = env.BUILD_NUMBER
                    //object.member
                    //object.property
                    //object.method()
                    def buildNumberInt = buildNumberStr.toInteger() //Convert string to integer
                    echo "Converted Build Number (Integer): ${buildNumberInt} (Type: ${buildNumberInt.getClass().getName()})"
                }
            }
        }

        stage("test"){
            steps{
                script{
                    def currentDate = java.time.LocalDate.now()
                    def formatter = java.time.format.DateTimeFormatter.ofPattern("MMM-dd-yyyy")
                    def formattedDate = currentDate.format(DateTimeFormatter)

                    echo "Formatted Date is : ${formattedDate}"

                    env.FORMATTED_DATE = formattedDate
                }
            }
        }

        stage('SSL Provisioning'){
            steps {
                script {
                    def dateString = "${env.FORMATTED_DATE}-v${env.BUILD_NUMBER}""
                    sh "echo \"${dateString}-v${env.BUILD_NUMBER} IN A 65.108.149.169\" | docker exec -i ubuntu-container tee -a /etc/coredns/zones/vishalmahawar.shop.db > /dev/null"
                }
            }
        }

        stage('Deploy to Another Server'){
            steps{
                sshagent (credentials: ['ID_RSA']) {
                    script{
                        def imageTag = "v${env.BUILD_NUMBER}";
                        //TypeCasting is the process to convert one data to another datatype
                        def hostPort = 8000 + env.BUILD_NUMBER.toInteger(); 
                        //po.then().then().then().catch().finally()
                        sh """
                            hostname && hostname -I
                            ssh -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_HOST '
                            hostname && hostname -I
                            docker pull ${DOCKER_IMAGE}:${imageTag}
                            docker run -d -p ${hostPort}:80 ${DOCKER_IMAGE}:${imageTag} /usr/sbin/apache2ctl -D FOREGROUND
                        '
                        hostname && hostname -I
                        """
                    }
                }
            }
        }
    }
}