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

        stage('Format Date') {
            steps {
                script {
                    def currentDate = java.time.LocalDate.now()
                    def formatter = java.time.format.DateTimeFormatter.ofPattern("MMM-dd-yyyy")
                    def formattedDate = currentDate.format(formatter)

                    def dateParts = formattedDate.split("-")
                    def month = dateParts[0].toLowerCase()
                    def finalDate = formattedDate.replaceFirst(dateParts[0], month)

                    // Save to environment variables for use in the next stage
                    env.D_DATE = finalDate
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

        stage('SSL Settingup'){
            steps{
                sshagent(credentials: ['ID_RSA']) {
                    script{
                        def sslDomain = "${env.D_DATE}-v${env.BUILD_NUMBER}.vishalmahawar.shop"
                        sh """
                            hostname && hostname -I
                            ssh -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_HOST 'hostname && hostname -I'
                            ssh -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_HOST 'apt update -y'
                            ssh -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_HOST 'apt upgrade -y'
                            ssh -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_HOST 'apt install -y sudo -y'
                            ssh -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_HOST 'sudo systemctl stop ufw && systemctl disable ufw'
                            ssh -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_HOST 'sudo apt install -y certbot python3-certbot-apache'
                            ssh -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_HOST 'certbot --version'                        
                        """
                        sh """
                            mkdir -p /etc/apache2/sites-available/${sslDomain}.conf
                            #3. Generate the vhost file dynamically
                            cat <<VHOST | tee /etc/apache2/sites-available/${sslDomain}.conf
<VirtualHost *:80>
    ServerName ${sslDomain}

    RewriteEngine on
    RewriteCond %{SERVER_NAME} =${sslDomain}
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]

    ProxyPreserveHost On
    ProxyPass / http://localhost:8065/
    ProxyPassReverse / http://localhost:8065/
</VirtualHost>
VHOST
                        # 4. Enable the new site and reload Apache
                        sudo a2ensite ${sslDomain}.conf
                        sudo systemctl reload apache2

                          # 5. Obtain (and install) the SSL certificate non-interactively
                        sudo certbot --apache -d ${sslDomain}

                         # 6. Confirm renewal job exists
                         sudo systemctl list-timers | grep certbot || true
                        """
                    }
                }
            }
        }
    }
}
