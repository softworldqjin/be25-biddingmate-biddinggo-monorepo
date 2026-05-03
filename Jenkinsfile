pipeline {
    agent any

    environment {
        SPRING_PROFILES_ACTIVE = 'local'
        LOCAL_PORT = '8080'
        DB_HOST = '127.0.0.1'
        DB_PORT = '3306'
        DB_NAME = 'biddinggo'
        DB_USERNAME = 'biddinggo'
        DB_PASSWORD = 'biddinggo'
        REDIS_HOST = '127.0.0.1'
        REDIS_PORT = '6379'
        REDIS_USERNAME = ''
        REDIS_PASSWORD = ''
        KAKAO_REST_API_KEY = 'test'
        KAKAO_CLIENT_SECRET = 'test'
        GOOGLE_CLIENT_ID = 'test'
        GOOGLE_CLIENT_SECRET = 'test'
        TOSS_SECRET_KEY = 'test'
        R2_ACCESS_KEY = 'test'
        R2_SECRET_KEY = 'test'
        R2_ACCOUNT_ID = 'test'
        R2_BUCKET = 'test'
        R2_PUBLIC_BASE_URL = 'https://example.com'
        JWT_SECRET_KEY = 'test-jwt-secret-key-for-jenkins'
        FRONTEND_REDIRECT_URI = 'http://localhost:5173/auth/callback'
        OPENAI_API_KEY = 'test'
        SUPABASE_URL = 'https://example.supabase.co'
        SUPABASE_SERVICE_ROLE_KEY = 'test'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Start CI Dependencies') {
            steps {
                sh 'docker compose -f docker-compose.ci.yml up -d'
                sh 'until [ "$(docker inspect -f {{.State.Health.Status}} biddinggo-ci-mariadb)" = "healthy" ]; do sleep 3; done'
                sh 'until [ "$(docker inspect -f {{.State.Health.Status}} biddinggo-ci-redis)" = "healthy" ]; do sleep 3; done'
            }
        }

        stage('Backend Test') {
            steps {
                dir('backend') {
                    sh 'chmod +x ./gradlew'
                    sh './gradlew clean test --no-daemon'
                }
            }
        }

        stage('Frontend Build') {
            steps {
                dir('frontend') {
                    sh 'npm ci'
                    sh 'npm run build'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('sonarqube-server') {
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }

    post {
        always {
            sh 'docker compose -f docker-compose.ci.yml down -v'
        }
    }
}
