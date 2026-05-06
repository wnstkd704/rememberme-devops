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
    }

    stages {
        stage('Detect Changes') {
            steps {
                script {
                    // 현재 커밋과 이전 커밋(HEAD~1) 간의 변경 파일을 가져온다.
                    def changedFiles = sh(script: 'git diff --name-only HEAD~1', returnStdout: true).trim().split("\n")

                    // 전체 배열을 줄바꿈으로 출력
                    echo "Changed files:\n${changedFiles.join('\n')}"
                    
                    // 환경 변수 동적 설정
                    env.SHOULD_BUILD_APP = changedFiles.any { it.startsWith("Frontend/") } ? "true" : "false"
                    env.SHOULD_BUILD_API = changedFiles.any { it.startsWith("Backend/") } ? "true" : "false"

                    echo "SHOULD_BUILD_APP : ${SHOULD_BUILD_APP}"
                    echo "SHOULD_BUILD_API : ${SHOULD_BUILD_API}"
                }
            }
        }

        stage('Docker Login') {
            when {
                expression { 
                    return env.SHOULD_BUILD_APP == "true" ||  env.SHOULD_BUILD_API == "true"
                }
            }

            steps {
                container('docker') {
                    sh 'docker logout'

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
                                sh 'echo $FRONTEND_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
                                sh 'docker build --no-cache -t $FRONTEND_IMAGE_NAME:$DOCKER_IMAGE_VERSION ./'
                                sh 'docker image inspect $FRONTEND_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
                                sh 'docker push $FRONTEND_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
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
                                sh 'echo $BACKEND_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
                                sh 'docker build --no-cache -t $BACKEND_IMAGE_NAME:$DOCKER_IMAGE_VERSION ./'
                                sh 'docker image inspect $BACKEND_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
                                sh 'docker push $BACKEND_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
                            }
                        }
                    }
                }
            }
        }

        stage('Trigger k8s-manifests') {
            steps {
                script {
                    def buildNumber = "${env.BUILD_NUMBER}"

                    withEnv(["DOCKER_IMAGE_VERSION=${buildNumber}"]) {
                        build job: 'rememberme-devops',
                            parameters: [
                                string(name: 'DOCKER_IMAGE_VERSION', value: "${DOCKER_IMAGE_VERSION}"),
                                string(name: 'DID_BUILD_APP', value: "${env.SHOULD_BUILD_APP}"),
                                string(name: 'DID_BUILD_API', value: "${env.SHOULD_BUILD_API}")
                            ],
                            wait: true
                    }
                }
            }
        }
    }
    
}pipeline {
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
        GITHUB_CREDENTIALS_ID = 'github-k8s-manifests' // SSH Agent ID
    }

    stages {
        stage('Detect Changes') {
            steps {
                script {
                    // Git 대소문자 이슈 방지를 위한 초기화 (선택 사항)
                    sh 'git config core.ignorecase false'
                    
                    // 현재 커밋과 이전 커밋 간의 변경 파일 확인
                    def changedFiles = sh(script: 'git diff --name-only HEAD~1', returnStdout: true).trim().split("\n")
                    echo "Changed files:\n${changedFiles.join('\n')}"
                    
                    // 빌드 여부 결정
                    env.SHOULD_BUILD_APP = changedFiles.any { it.startsWith("Frontend/") } ? "true" : "false"
                    env.SHOULD_BUILD_API = changedFiles.any { it.startsWith("Backend/") } ? "true" : "false"

                    echo "SHOULD_BUILD_APP : ${env.SHOULD_BUILD_APP}"
                    echo "SHOULD_BUILD_API : ${env.SHOULD_BUILD_API}"
                }
            }
        }

        stage('Docker Login') {
            when {
                expression { return env.SHOULD_BUILD_APP == "true" || env.SHOULD_BUILD_API == "true" }
            }
            steps {
                container('docker') {
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

        stage('Frontend Build & Push') {
            when { expression { return env.SHOULD_BUILD_APP == "true" } }
            steps {
                container('docker') {
                    dir('Frontend') {
                        script {
                            sh "docker build --no-cache -t ${FRONTEND_IMAGE_NAME}:${env.BUILD_NUMBER} ./"
                            sh "docker push ${FRONTEND_IMAGE_NAME}:${env.BUILD_NUMBER}"
                        }
                    }
                }
            }
        }

        stage('Backend Build & Push') {
            when { expression { return env.SHOULD_BUILD_API == "true" } }
            steps {
                container('docker') {
                    dir('Backend') {
                        script {
                            sh "docker build --no-cache -t ${BACKEND_IMAGE_NAME}:${env.BUILD_NUMBER} ./"
                            sh "docker push ${BACKEND_IMAGE_NAME}:${env.BUILD_NUMBER}"
                        }
                    }
                }
            }
        }

        stage('Update K8s Manifests') {
            when {
                expression { return env.SHOULD_BUILD_APP == "true" || env.SHOULD_BUILD_API == "true" }
            }
            steps {
                script {
                    // 이미지 구조(image_693e70.png)에 맞춘 경로 수정
                    if (env.SHOULD_BUILD_APP == "true") {
                        sh "sed -i 's|${FRONTEND_IMAGE_NAME}:.*|${FRONTEND_IMAGE_NAME}:${env.BUILD_NUMBER}|g' k8s/frontend/deployment.yaml"
                        sh "cat k8s/frontend/deployment.yaml"
                    }
                    
                    if (env.SHOULD_BUILD_API == "true") {
                        sh "sed -i 's|${BACKEND_IMAGE_NAME}:.*|${BACKEND_IMAGE_NAME}:${env.BUILD_NUMBER}|g' k8s/backend/deployment.yaml"
                        sh "cat k8s/backend/deployment.yaml"
                    }
                }
            }
        }

        stage('Commit & Push to Git') {
            when {
                expression { return env.SHOULD_BUILD_APP == "true" || env.SHOULD_BUILD_API == "true" }
            }
            steps {
                sh 'git config user.name "jenkins"'
                sh 'git config user.email "jenkins@beyond.com"'
                sh "git add k8s/"
                sh "git commit -m 'Update Image Version ${env.BUILD_NUMBER}'"
                
                sshagent([GITHUB_CREDENTIALS_ID]) {
                    sh 'git push origin main'
                }
            }
        }
    }
}