pipeline {
    agent any
    environment {
        DOCKER_IMAGE = "vishalmahawar5200/10may2025"
        DEPLOY_USER = "root"
        DEPLOY_HOST = "65.108.149.166"
        SSL_DOMAIN = "${env.D_DATE}-v${env.BUILD_NUMBER}.vishalmahawar.shop"
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

        stage('Check Build Number') {
            steps {
                script {
                    def buildNumberStr = env.BUILD_NUMBER
                    def buildNumberInt = buildNumberStr.toInteger() // Convert string to integer
                    echo "Converted Build Number (Integer): ${buildNumberInt} (Type: ${buildNumberInt.getClass().getName()})"
                }
            }
        }

        stage('Formate Date') {
            steps {
                script {
                    // Set 'today' once and reuse in all stages
                    today = new Date().format("MMM-dd-yyyy", TimeZone.getTimeZone('UTC')).toLowerCase()
                    echo "Today's tag: ${today}"
                }
            }
        }

        stage('SSL Provisioning') {
            steps {
                script {
                    // Ensure D_DATE is properly referenced
                    def dateString = "${env.D_DATE}-v${env.BUILD_NUMBER}"
                    sh """
                        echo \$(date +%B-%d-%Y | tr '[:upper:]' '[:lower:]')-v${env.BUILD_NUMBER} IN A 65.108.149.166 | docker exec -i ubuntu-container tee -a /etc/coredns/zones/vishalmahawar.shop.db > /dev/null
                    """
                }
            }
        }

        stage('Deploy to Another Server') {
            steps {
                sshagent(credentials: ['ID_RSA']) {
                    script {
                        def imageTag = "v${env.BUILD_NUMBER}"
                        def hostPort = 8000 + env.BUILD_NUMBER.toInteger()
                        sh """
                            ssh -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_HOST '
                            docker pull ${DOCKER_IMAGE}:${imageTag}
                            docker run -d -p ${hostPort}:80 ${DOCKER_IMAGE}:${imageTag} /usr/sbin/apache2ctl -D FOREGROUND
                            '
                        """
                    }
                }
            }
        }

         stage("Configure Apache & SSL") {
            steps {
                script {
                    def subdomain = "${today}-v${BUILD_NUMBER}.vishalmahawar.shop"
                    def hostPort = 8000 + env.BUILD_NUMBER.toInteger()

                    sshagent(credentials: ['ID_RSA']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_HOST <<EOF
cat > /etc/apache2/sites-available/${subdomain}.conf <<EOL
<VirtualHost *:80>
    ServerName ${subdomain}

    ProxyPreserveHost On
    ProxyPass / http://localhost:${port}/
    ProxyPassReverse / http://localhost:${port}/

    ErrorLog \${APACHE_LOG_DIR}/${subdomain}_error.log
    CustomLog \${APACHE_LOG_DIR}/${subdomain}_access.log combined
</VirtualHost>
EOL

a2ensite ${subdomain}
systemctl reload apache2

echo "Waiting 120 seconds for DNS to propagate..."
sleep 120

certbot --apache -d ${subdomain} --agree-tos -m vishalmahawar.shop@gmail.com --non-interactive
EOF
                        """
                    }
                }
            }
        }

        stage('Restart Jenkins Container'){
            steps{
                sshagent(credentials: ['ID_RSA']){
                    sh """
                        ssh -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_HOST
                        'docker restart 780b2234ce7b && docker ps -a'
                    """
                }
            }
        }
    }
}
