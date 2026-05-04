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
              - name: docker
                image: docker:29.4.1-cli-alpine3.23
                command:
                - cat
                tty: true
                volumeMounts:
                - mountPath: "/var/run/docker.sock"
                  name: docker-socket
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
              volumes:
              - name: docker-socket
                hostPath:
                  path: "/var/run/docker.sock"
            '''
        }
    }

    environment {
        DOCKER_CREDENTIALS_ID = 'dockerhub-access'
        BACKEND_IMAGE_NAME = 'gyujin123/biddinggo-backend'
        FRONTEND_IMAGE_NAME = 'gyujin123/biddinggo-frontend'
        BUILD_FRONT = 'false'
        BUILD_BACK = 'false'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Detect Changes') {
            steps {
                script {
                    def changedOutput = sh(
                        script: '''
                            if git rev-parse --verify HEAD~1 >/dev/null 2>&1; then
                                git diff --name-only HEAD~1 HEAD
                            else
                                git ls-files
                            fi
                        ''',
                        returnStdout: true
                    ).trim()

                    def changedFiles = changedOutput ? changedOutput.split("\\n") : []

                    env.BUILD_FRONT = changedFiles.any { it.startsWith('frontend/') } ? 'true' : 'false'
                    env.BUILD_BACK = changedFiles.any { it.startsWith('backend/') } ? 'true' : 'false'

                    echo "Changed files:\n${changedOutput ?: '(none)'}"
                    echo "BUILD_FRONT=${env.BUILD_FRONT}"
                    echo "BUILD_BACK=${env.BUILD_BACK}"

                    if (env.BUILD_FRONT == 'false' && env.BUILD_BACK == 'false') {
                        echo 'No frontend or backend changes detected. Build and push stages will be skipped.'
                    }
                }
            }
        }

        stage('Frontend Build') {
            when {
                expression { env.BUILD_FRONT == 'true' }
            }
            steps {
                container('node') {
                    dir('frontend') {
                        withCredentials([file(credentialsId: 'frontend-ci-env', variable: 'FRONTEND_ENV_FILE')]) {
                            sh '''
                                set +x
                                trap 'rm -f .env' EXIT
                                cp "$FRONTEND_ENV_FILE" .env
                                chmod 600 .env
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

        stage('Backend Build') {
            when {
                expression { env.BUILD_BACK == 'true' }
            }
            steps {
                container('mariadb') {
                    sh 'until mariadb-admin ping -h 127.0.0.1 -u root -p"$MARIADB_ROOT_PASSWORD" --silent; do sleep 3; done'
                }
                container('redis') {
                    sh 'until redis-cli -h 127.0.0.1 ping | grep PONG; do sleep 3; done'
                }
                container('gradle') {
                    dir('backend') {
                        withCredentials([file(credentialsId: 'backend-ci-env', variable: 'BACKEND_ENV_FILE')]) {
                            sh '''
                                set +x
                                trap 'rm -f .env' EXIT
                                cp "$BACKEND_ENV_FILE" .env
                                chmod 600 .env
                                chmod +x ./gradlew
                                SPRING_PROFILES_ACTIVE=local ./gradlew clean build --no-daemon
                            '''
                        }
                    }
                }
            }
        }

        stage('Docker Image Build & Push') {
            when {
                expression { env.BUILD_FRONT == 'true' || env.BUILD_BACK == 'true' }
            }
            steps {
                container('docker') {
                    script {
                        def imageVersion = "${env.BUILD_NUMBER}"

                        withCredentials([usernamePassword(
                            credentialsId: DOCKER_CREDENTIALS_ID,
                            usernameVariable: 'DOCKER_USERNAME',
                            passwordVariable: 'DOCKER_PASSWORD'
                        )]) {
                            sh '''
                                set +x
                                echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                            '''
                        }

                        try {
                            if (env.BUILD_BACK == 'true') {
                                sh """
                                    docker build -t ${BACKEND_IMAGE_NAME}:${imageVersion} backend
                                    docker push ${BACKEND_IMAGE_NAME}:${imageVersion}
                                """
                            }

                            if (env.BUILD_FRONT == 'true') {
                                sh """
                                    docker build -t ${FRONTEND_IMAGE_NAME}:${imageVersion} frontend
                                    docker push ${FRONTEND_IMAGE_NAME}:${imageVersion}
                                """
                            }
                        } finally {
                            sh 'docker logout || true'
                        }
                    }
                }
            }
        }
    }

}
