
def notifyTelegram(String status) {
    withCredentials([
        string(credentialsId: 'telegram-bot-token',
               variable: 'TG_TOKEN'),
        string(credentialsId: 'telegram-chat-id',
               variable: 'TG_CHAT_ID')
    ]) {
        withEnv(["BUILD_STATUS=${status}"]) {
            sh '''
                set +x
                curl --fail --silent --show-error \
                  --request POST \
                  --data-urlencode "chat_id=$TG_CHAT_ID" \
                  --data-urlencode "text=La Shop CI/CD: $BUILD_STATUS - Build #$BUILD_NUMBER" \
                  "https://api.telegram.org/bot$TG_TOKEN/sendMessage" > /dev/null
            '''
        }
    }
}

pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    stages {
        stage('Checkout Frontend') {
            steps {
                dir('frontend') {
                    git branch: 'main',
                        url: 'https://github.com/hoangphu11099/Lashop-Fontend.git'
                }
            }
        }

        stage('Checkout Backend') {
            steps {
                dir('backend') {
                    git branch: 'main',
                        url: 'https://github.com/hoangphu11099/Lashop-Backend.git'
                }
            }
        }

        stage('Frontend CI') {
            steps {
                dir('frontend') {
                    sh 'npm ci'
                     sh 'npm run lint'
                    sh 'npm run build'
                }
            }
        }

        stage('Backend CI') {
            steps {
                dir('backend') {
                    sh 'go test ./...'
                    sh 'go build ./...'
                }
            }
        }

        stage('Docker Build Windows') {
            agent {
                label 'lashop-docker'
            }

            steps {
                dir('frontend') {
                    git branch: 'main',
                        url: 'https://github.com/hoangphu11099/Lashop-Fontend.git'

                    bat '''
                        docker buildx build --builder lashop-builder --platform linux/amd64 -t lashop-frontend:ci-test --load .
                    '''
                }

                dir('backend') {
                    git branch: 'main',
                        url: 'https://github.com/hoangphu11099/Lashop-Backend.git'

                    bat '''
                        docker buildx build --builder lashop-builder --platform linux/amd64 -t lashop-backend:ci-test --load .
                    '''
                }

                bat '''
                    docker save -o lashop-images.tar lashop-frontend:ci-test lashop-backend:ci-test
                '''

                stash name: 'lashop-images',
                      includes: 'lashop-images.tar'
            }
        }

        stage('Upload Images VPS') {
            steps {
                unstash 'lashop-images'

                sshagent(credentials: ['lashop-vps-ssh-new']) {
                    sh '''
                        scp -o BatchMode=yes \
                            -o StrictHostKeyChecking=yes \
                            -P 24700 \
                            lashop-images.tar \
                            deploy@103.166.183.24:/home/deploy/lashop-images.tar
                    '''
                }
            }
        }

        stage('Deploy VPS') {
            steps {
                sshagent(credentials: ['lashop-vps-ssh-new']) {
                    sh '''
                        ssh -o BatchMode=yes \
                            -o StrictHostKeyChecking=yes \
                            -p 24700 \
                            deploy@103.166.183.24 \
                            'sudo -n /usr/local/sbin/lashop-deploy'
                    '''
                }
            }
        }

        stage('Website Health Check') {
            steps {
                sh '''
                    curl --fail --silent --show-error \
                      https://lashop.hoangphu.click/api/health
                '''
            }
        }
    }

    post {
        success {
            script {
                notifyTelegram('SUCCESS')
            }
        }

        failure {
            script {
                notifyTelegram('FAILURE')
            }
        }
    }
}
