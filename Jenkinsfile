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
        DOCKER_CREDENTIALS_ID = 'dockerhub-access'
        GITHUB_CREDENTIALS_ID = 'github-k8s-manifests'
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

                    env.SHOULD_BUILD_FRONTEND =
                        changedFiles.any { it.startsWith("Frontend/") }
                        ? "true" : "false"

                    env.SHOULD_BUILD_BACKEND =
                        changedFiles.any { it.startsWith("Backend/") }
                        ? "true" : "false"

                    echo "SHOULD_BUILD_FRONTEND : ${env.SHOULD_BUILD_FRONTEND}"
                    echo "SHOULD_BUILD_BACKEND : ${env.SHOULD_BUILD_BACKEND}"
                }
            }
        }

        stage('Docker Login') {

            when {
                expression {
                    return env.SHOULD_BUILD_FRONTEND == "true" ||
                           env.SHOULD_BUILD_BACKEND == "true"
                }
            }

            steps {
                container('docker') {

                    withCredentials([usernamePassword(
                        credentialsId: DOCKER_CREDENTIALS_ID,
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )]) {

                        sh '''
                        echo $DOCKER_PASSWORD | \
                        docker login -u $DOCKER_USERNAME --password-stdin
                        '''
                    }
                }
            }
        }

        stage('Frontend Build & Push') {

            when {
                expression {
                    return env.SHOULD_BUILD_FRONTEND == "true"
                }
            }

            steps {
                container('docker') {

                    dir('Frontend') {

                        sh """
                        docker build --no-cache \
                          -t ${FRONTEND_IMAGE_NAME}:${BUILD_NUMBER} \
                          -t ${FRONTEND_IMAGE_NAME}:latest \
                          .
                        """

                        sh "docker push ${FRONTEND_IMAGE_NAME}:${BUILD_NUMBER}"
                        sh "docker push ${FRONTEND_IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Backend Build & Push') {

            when {
                expression {
                    return env.SHOULD_BUILD_BACKEND == "true"
                }
            }

            steps {
                container('docker') {

                    dir('Backend') {

                        sh """
                        docker build --no-cache \
                          --build-arg BUILD_PROFILE=local \
                          --build-arg BUILD_PORT=8088 \
                          -t ${BACKEND_IMAGE_NAME}:${BUILD_NUMBER} \
                          -t ${BACKEND_IMAGE_NAME}:latest \
                          .
                        """

                        sh "docker push ${BACKEND_IMAGE_NAME}:${BUILD_NUMBER}"
                        sh "docker push ${BACKEND_IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Update K8s Manifest') {

            when {
                expression {
                    return env.SHOULD_BUILD_FRONTEND == "true" ||
                           env.SHOULD_BUILD_BACKEND == "true"
                }
            }

            steps {
                container('docker') {

                    script {

                        if (env.SHOULD_BUILD_FRONTEND == "true") {

                            sh """
                            sed -i 's|image: ${FRONTEND_IMAGE_NAME}:.*|image: ${FRONTEND_IMAGE_NAME}:${BUILD_NUMBER}|g' \
                            k8s/frontend/deployment.yaml
                            """
                        }

                        if (env.SHOULD_BUILD_BACKEND == "true") {

                            sh """
                            sed -i 's|image: ${BACKEND_IMAGE_NAME}:.*|image: ${BACKEND_IMAGE_NAME}:${BUILD_NUMBER}|g' \
                            k8s/backend/deployment.yaml
                            """
                        }
                    }
                }
            }
        }

        stage('Commit & Push') {

            when {
                expression {
                    return env.SHOULD_BUILD_FRONTEND == "true" || env.SHOULD_BUILD_BACKEND == "true"
                }
            }

            steps {

                sh 'git config user.name "jenkins"'
                sh 'git config user.email "jenkins@beyond.com"'

                sh 'git add k8s/'

                sh '''
                git diff --cached --quiet || \
                git commit -m "chore: update image version ${BUILD_NUMBER}"
                '''

                sshagent([GITHUB_CREDENTIALS_ID]) {
                    sh 'git push origin main'
                }
            }
        }
    }
}