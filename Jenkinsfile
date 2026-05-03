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
                        withCredentials([file(credentialsId: 'backend-ci-env', variable: 'BACKEND_ENV_FILE')]) {
                            sh '''
                                set -a
                                . "$BACKEND_ENV_FILE"
                                set +a
                                chmod +x ./gradlew
                                ./gradlew clean test --no-daemon
                            '''
                        }
                    }
                }
            }
        }

        stage('Frontend Build') {
            steps {
                container('node') {
                    dir('frontend') {
                        withCredentials([file(credentialsId: 'frontend-ci-env', variable: 'FRONTEND_ENV_FILE')]) {
                            sh '''
                                cp "$FRONTEND_ENV_FILE" .env
                                node -v
                                npm -v
                                npm ci
                                npm run build
                            '''
                        }
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
