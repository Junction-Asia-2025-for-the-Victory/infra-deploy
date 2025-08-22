pipeline {
    agent any
    options {
        timestamps()
    }
    parameters {
        string(name: 'BACKEND_TAG', defaultValue: 'latest')
        string(name: 'FRONTEND_TAG', defaultValue: 'latest')
    }
    environment {
        DC = 'docker compose'
        // 로컬 이미지 이름 사용
        BACKEND_IMAGE = "victory-backend:${params.BACKEND_TAG}"
        FRONTEND_IMAGE = "victory-frontend:${params.FRONTEND_TAG}"
    }
    stages {
        stage('Checkout'){
            steps {
                checkout scm
            }
        }
        stage('Verify Local Images'){
            steps {
                sh """
                    echo "=== Checking if local images exist ==="
                    echo "Backend image: ${BACKEND_IMAGE}"
                    echo "Frontend image: ${FRONTEND_IMAGE}"
                    
                    # 이미지 존재 확인
                    if docker images ${BACKEND_IMAGE} --format "table {{.Repository}}:{{.Tag}}" | grep -v REPOSITORY | grep -q .; then
                        echo "✅ Backend image found: ${BACKEND_IMAGE}"
                    else
                        echo "❌ Backend image not found: ${BACKEND_IMAGE}"
                        echo "Available victory-backend images:"
                        docker images victory-backend || echo "No victory-backend images found"
                        exit 1
                    fi
                    
                    if docker images ${FRONTEND_IMAGE} --format "table {{.Repository}}:{{.Tag}}" | grep -v REPOSITORY | grep -q .; then
                        echo "✅ Frontend image found: ${FRONTEND_IMAGE}"
                    else
                        echo "❌ Frontend image not found: ${FRONTEND_IMAGE}"
                        echo "Available victory-frontend images:"
                        docker images victory-frontend || echo "No victory-frontend images found"
                        exit 1
                    fi
                """
            }
        }
        stage('Render .env from Credentials'){
            steps {
                withCredentials([
                    string(credentialsId: 'GOOGLE_CLIENT_ID', variable: 'GOOGLE_CLIENT_ID'),
                    string(credentialsId: 'GOOGLE_CLIENT_SECRET', variable: 'GOOGLE_CLIENT_SECRET'),
                    string(credentialsId: 'GOOGLE_REDIRECT_URL', variable: 'GOOGLE_REDIRECT_URL'),
                    string(credentialsId: 'JWT_SECRET_KEY', variable: 'JWT_SECRET_KEY'),
                    string(credentialsId: 'DB_URL', variable: 'DB_URL'),
                    string(credentialsId: 'DB_USERNAME', variable: 'DB_USERNAME'),
                    string(credentialsId: 'DB_PASSWORD', variable: 'DB_PASSWORD'),
                    string(credentialsId: 'FRONTEND_URL', variable: 'FRONTEND_URL'),
                    string(credentialsId: 'MYSQL_DATABASE', variable: 'MYSQL_DATABASE'),
                    string(credentialsId: 'MYSQL_USER', variable: 'MYSQL_USER'),
                    string(credentialsId: 'MYSQL_PASSWORD', variable: 'MYSQL_PASSWORD'),
                    string(credentialsId: 'MYSQL_ROOT_PASSWORD', variable: 'MYSQL_ROOT_PASSWORD'),
                    string(credentialsId: 'AES_SECRET_KEY', variable: 'AES_SECRET_KEY')
                ]) {
                    sh """
                        echo "=== Creating .env file ==="
                        cat > .env <<EOF
BACKEND_IMAGE=${BACKEND_IMAGE}
FRONTEND_IMAGE=${FRONTEND_IMAGE}
GOOGLE_CLIENT_ID=\${GOOGLE_CLIENT_ID}
GOOGLE_CLIENT_SECRET=\${GOOGLE_CLIENT_SECRET}
GOOGLE_REDIRECT_URL=\${GOOGLE_REDIRECT_URL}
JWT_SECRET_KEY=\${JWT_SECRET_KEY}
DB_URL=\${DB_URL}
DB_USERNAME=\${DB_USERNAME}
DB_PASSWORD=\${DB_PASSWORD}
FRONTEND_URL=\${FRONTEND_URL}
MYSQL_DATABASE=\${MYSQL_DATABASE}
MYSQL_USER=\${MYSQL_USER}
MYSQL_PASSWORD=\${MYSQL_PASSWORD}
MYSQL_ROOT_PASSWORD=\${MYSQL_ROOT_PASSWORD}
AES_SECRET_KEY=\${AES_SECRET_KEY}
EOF
                        
                        echo "=== Verifying .env file ==="
                        ls -la .env
                        echo "File created successfully"
                    """
                }
            }
        }
        stage('Deploy'){
            steps {
                sh """
                    echo "=== Starting deployment with local images ==="
                    echo "Backend image: ${BACKEND_IMAGE}"
                    echo "Frontend image: ${FRONTEND_IMAGE}"
                    
                    # 로컬 이미지이므로 pull 생략
                    ${DC} up -d
                    ${DC} ps
                """
            }
        }
        stage('Deploy Nginx Config') {
            steps {
                script {
                    // nginx 설정 파일 백업
                    sh 'sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/default.backup.$(date +%Y%m%d_%H%M%S)'
                    
                    // 새 설정 파일 복사
                    sh 'sudo cp nginx/nginx.conf /etc/nginx/sites-available/default'
                    
                    // nginx 설정 문법 확인
                    sh 'sudo nginx -t'
                    
                    // nginx 재로드
                    sh 'sudo systemctl reload nginx'
                    
                    echo 'Nginx configuration updated successfully'
                }
            }
        }
    }
    post {
        success {
            echo "✅ Deployed successfully with local images!"
        }
        failure {
            echo "❌ Deploy failed"
            sh """
                (${DC} logs --no-color --tail 50 mysql backend frontend || true) > deploy_failure.log
            """
        }
        always {
            sh 'rm -f .env || true'
            archiveArtifacts artifacts: 'deploy_failure.log', onlyIfSuccessful: false, allowEmptyArchive: true
        }
    }
}