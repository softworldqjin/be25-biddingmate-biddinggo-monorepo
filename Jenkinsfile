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
              - name: docker
                image: docker:29.4.1-cli-alpine3.23
                command:
                - cat
                tty: true
                volumeMounts:
                - mountPath: "/var/run/docker.sock"
                  name: docker-socket
              volumes:
              - name: docker-socket
                hostPath:
                  path: "/var/run/docker.sock"
            '''
        }
    }

    environment {
        SONARQUBE_SERVER = 'sonarqube-server'
        DOCKER_COMPOSE_FILE = 'docker-compose.ci.yml'

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
                container('docker') {
                    sh 'docker version'
                    sh 'docker compose version'
                    sh 'docker compose -f $DOCKER_COMPOSE_FILE up -d'
                    sh 'until [ "$(docker inspect -f {{.State.Health.Status}} biddinggo-ci-mariadb)" = "healthy" ]; do sleep 3; done'
                    sh 'until [ "$(docker inspect -f {{.State.Health.Status}} biddinggo-ci-redis)" = "healthy" ]; do sleep 3; done'
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
                }
            }
        }
    }

    post {
        always {
            container('docker') {
                sh 'docker compose -f $DOCKER_COMPOSE_FILE down -v || true'
            }
        }
    }
}
