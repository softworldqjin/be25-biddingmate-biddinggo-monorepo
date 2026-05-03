pipeline {
    agent {
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            metadata:
              name: biddinggo-jenkins-agent
            spec:
              containers:
              - name: gradle
                image: gradle:8.14.3-jdk21-alpine
                command:
                - cat
                tty: true
              - name: node
                image: node:22-alpine
                command:
                - cat
                tty: true
              - name: sonar
                image: sonarsource/sonar-scanner-cli:latest
                command:
                - cat
                tty: true
              - name: mariadb
                image: mariadb:11.8.5
                env:
                - name: MARIADB_ROOT_PASSWORD
                  value: biddinggo
                - name: MARIADB_DATABASE
                  value: biddinggo
                - name: MARIADB_USER
                  value: biddinggo
                - name: MARIADB_PASSWORD
                  value: biddinggo
              - name: redis
                image: redis:7.4
            '''
        }
    }

    environment {
        SONARQUBE_SERVER = 'sonarqube-server'

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

        stage('Wait For Dependencies') {
            steps {
                container('mariadb') {
                    sh 'until mariadb-admin ping -h 127.0.0.1 -u root -p"$MARIADB_ROOT_PASSWORD" --silent; do sleep 3; done'
                }
                container('redis') {
                    sh 'until redis-cli -h 127.0.0.1 ping | grep PONG; do sleep 3; done'
                }
            }
        }

        stage('Backend Test') {
            steps {
                container('gradle') {
                    dir('backend') {
                        sh 'chmod +x ./gradlew'
                        sh './gradlew clean test --no-daemon'
                    }
                }
            }
        }

        stage('Frontend Build') {
            steps {
                container('node') {
                    dir('frontend') {
                        sh 'node -v'
                        sh 'npm -v'
                        sh 'npm ci'
                        sh 'npm run build'
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('sonar') {
                    withSonarQubeEnv("${SONARQUBE_SERVER}") {
                        sh 'sonar-scanner'
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                    //
                }
            }
        }
    }

}
