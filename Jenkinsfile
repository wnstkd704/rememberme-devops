pipeline {
    agent {
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            metadata:
              name: jenkins-agent
            spec:
              containers:
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
        BACKEND_IMAGE_NAME = 'junsang704/rememberme-backend'
        FRONTEND_IMAGE_NAME = 'junsang704/rememberme-frontend'
        DOCKER_CREDENTIALS_ID = 'dockerhub-access-rememberme'
        GITHUB_CREDENTIALS_ID = 'github-rememberme-devops'
    }

    stages {
        stage('Detect Changes') {
            steps {
                script {
                    def changedFiles = sh(
                        script: 'git diff --name-only HEAD~1',
                        returnStdout: true
                    ).trim().split("\n")

                    echo "Changed files:\n${changedFiles.join('\n')}"

                    env.SHOULD_BUILD_APP = changedFiles.any {
                        it.startsWith("Frontend/")
                    } ? "true" : "false"

                    env.SHOULD_BUILD_API = changedFiles.any {
                        it.startsWith("Backend/")
                    } ? "true" : "false"

                    echo "SHOULD_BUILD_APP : ${env.SHOULD_BUILD_APP}"
                    echo "SHOULD_BUILD_API : ${env.SHOULD_BUILD_API}"
                }
            }
        }

        stage('Docker Login') {
            when {
                expression {
                    return env.SHOULD_BUILD_APP == "true" ||
                            env.SHOULD_BUILD_API == "true"
                }
            }

            steps {
                container('docker') {
                    sh 'docker logout || true'

                    withCredentials([usernamePassword(
                        credentialsId: DOCKER_CREDENTIALS_ID,
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )]) {
                        sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                    }
                }
            }
        }

        stage('Frontend Image Build & Push') {
            when {
                expression {
                    return env.SHOULD_BUILD_APP == "true"
                }
            }

            steps {
                container('docker') {
                    dir('Frontend') {
                        script {
                            def buildNumber = "${env.BUILD_NUMBER}"

                            withEnv(["DOCKER_IMAGE_VERSION=${buildNumber}"]) {
                                sh 'docker -v'
                                sh 'echo $APP_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
                                sh 'docker build --no-cache -t $APP_IMAGE_NAME:$DOCKER_IMAGE_VERSION ./'
                                sh 'docker image inspect $APP_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
                                sh 'docker push $APP_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
                            }
                        }
                    }
                }
            }
        }

        stage('Backend Image Build & Push') {
            when {
                expression {
                    return env.SHOULD_BUILD_API == "true"
                }
            }

            steps {
                container('docker') {
                    dir('Backend') {
                        script {
                            def buildNumber = "${env.BUILD_NUMBER}"

                            withEnv(["DOCKER_IMAGE_VERSION=${buildNumber}"]) {
                                sh 'docker -v'
                                sh 'echo $API_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
                                sh 'docker build --no-cache -t $API_IMAGE_NAME:$DOCKER_IMAGE_VERSION ./'
                                sh 'docker image inspect $API_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
                                sh 'docker push $API_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
                            }
                        }
                    }
                }
            }
        }

        stage('Update k8s Image Tag') {
            when {
                expression {
                    return env.SHOULD_BUILD_APP == "true" ||
                        env.SHOULD_BUILD_API == "true"
                }
            }

            steps {
                script {

                    if (env.SHOULD_BUILD_APP == "true") {

                        echo "Update Frontend rollout.yaml"

                        sh """
                        sed -i 's|image: ${FRONTEND_IMAGE_NAME}:.*|image: ${FRONTEND_IMAGE_NAME}:${BUILD_NUMBER}|g' \
                        k8s/frontend/rollout.yaml
                        """

                        sh 'cat k8s/frontend/rollout.yaml'
                    }

                    if (env.SHOULD_BUILD_API == "true") {

                        echo "Update Backend rollout.yaml"

                        sh """
                        sed -i 's|image: ${BACKEND_IMAGE_NAME}:.*|image: ${BACKEND_IMAGE_NAME}:${BUILD_NUMBER}|g' \
                        k8s/backend/rollout.yaml
                        """

                        sh 'cat k8s/backend/rollout.yaml'
                    }
                }
            }
        }

        stage('Commit & Push k8s Changes') {
            when {
                expression {
                    return env.SHOULD_BUILD_APP == "true" ||
                        env.SHOULD_BUILD_API == "true"
                }
            }

            steps {
                sh 'git config user.name "jenkins"'
                sh 'git config user.email "jenkins@beyond.com"'

                sh 'git add k8s/'

                sh '''
                git diff --cached --quiet || \
                git commit -m "chore: update image version ${BUILD_NUMBER} [skip ci]"
                '''

                sshagent([GITHUB_CREDENTIALS_ID]) {
                    sh 'git checkout -b manifest'
                    sh 'git push origin manifest'
                    sh 'git checkout main'
                }
            }
        }
    }
}