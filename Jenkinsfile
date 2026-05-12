pipeline {
    agent any
    stages {
        stage('Check changes') {
            steps {
                sh '''
                    CHANGED=$(git diff HEAD~1 HEAD --name-only)
                    echo "Changed files: $CHANGED"
                    if ! echo "$CHANGED" | grep -q "index.html"; then
                        echo "index.html not changed, skipping..."
                        exit 0
                    fi
                    echo "index.html changed, continuing..."
                '''
            }
        }
        stage('Run nginx') {
            steps {
                sh '''
                    docker run -d \
                        --name nginx_test \
                        -p 9889:80 \
                        -v $(pwd)/index.html:/usr/share/nginx/html/index.html \
                        nginx
                    sleep 3
                '''
            }
        }
        stage('Check HTTP 200') {
            steps {
                sh '''
                    CODE=$(curl -o /dev/null -s -w "%{http_code}" http://localhost:9889)
                    if [ "$CODE" != "200" ]; then
                        echo "HTTP code is $CODE, expected 200"
                        exit 1
                    fi
                '''
            }
        }
        stage('Check MD5') {
            steps {
                sh '''
                    FILE_MD5=$(md5sum index.html | awk '{print $1}')
                    curl -s http://localhost:9889 > /tmp/nginx_response.html
                    NGINX_MD5=$(md5sum /tmp/nginx_response.html | awk '{print $1}')
                    echo "File MD5: $FILE_MD5"
                    echo "Nginx MD5: $NGINX_MD5"
                    if [ "$FILE_MD5" != "$NGINX_MD5" ]; then
                        echo "MD5 mismatch!"
                        exit 1
                    fi
                '''
            }
        }
    }
    post {
        success {
            withCredentials([
                string(credentialsId: 'telegram-token', variable: 'TOKEN'),
                string(credentialsId: 'telegram-chatid', variable: 'CHAT_ID')
            ]) {
                sh '''
                    curl -s -X POST https://api.telegram.org/bot${TOKEN}/sendMessage \
                        -d chat_id=${CHAT_ID} \
                        -d text="CI SUCCESS! Деплой прошёл успешно"
                '''
            }
        }
        failure {
            withCredentials([
                string(credentialsId: 'telegram-token', variable: 'TOKEN'),
                string(credentialsId: 'telegram-chatid', variable: 'CHAT_ID')
            ]) {
                sh '''
                    curl -s -X POST https://api.telegram.org/bot${TOKEN}/sendMessage \
                        -d chat_id=${CHAT_ID} \
                        -d text="CI FAILED! Проверь Jenkins"
                '''
            }
        }
        always {
            sh 'docker rm -f nginx_test || true'
        }
    }
}