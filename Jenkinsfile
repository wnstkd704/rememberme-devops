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

        stage('Backend Image Build & Push') {
            when {
                expression {
                    return env.SHOULD_BUILD_API == "true"
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
                    // 1. manifest 브랜치가 이미 있으면 체크아웃, 없으면 생성
                    sh 'git checkout manifest || git checkout -b manifest'
                    
                    // 2. 원격의 최신 내용을 가져와서 현재 커밋을 그 위로 재배치 (충돌 해결 핵심)
                    // 만약 원격에 브랜치가 아예 없다면 에러가 날 수 있으므로 || true 처리
                    sh 'git pull origin manifest --rebase || true'
                    
                    // 3. 푸시 시도
                    sh 'git push origin manifest'
                }
            }
        }
    }
}