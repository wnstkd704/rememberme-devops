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

    parameters {
        string(name: 'DOCKER_IMAGE_VERSION', defaultValue: '', description: 'Docker Image Version')
        string(name: 'DID_BUILD_APP', defaultValue: '', description: 'Did Build APP')
        string(name: 'DID_BUILD_API', defaultValue: '', description: 'Did Build API')
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

        stage('Checkout Main Branches') {
            steps {
                sh 'git checkout main'
                echo "DOCKER_IMAGE_VERSION: ${params.DOCKER_IMAGE_VERSION}"
                echo "DID_BUILD_APP: ${params.DID_BUILD_APP}"
                echo "DID_BUILD_API: ${params.DID_BUILD_API}"
            }
        }

        stage('update Vue rollout.yaml') {
            when {
                expression {
                    return params.DID_BUILD_APP == "true"
                }
            }

            steps {
                dir('university-vue') {
                    sh 'pwd'
                    sh 'ls -al'
                    echo "Received Docker Image Version : ${params.DOCKER_IMAGE_VERSION}"
                    sh "sed -i 's|junsang704/university-vue:.*|junsang704/university-vue:${params.DOCKER_IMAGE_VERSION}|g' rollout.yaml"
                    sh 'cat rollout.yaml'
                }
            }
        }

        stage('update API rollout.yaml') {
            when {
                expression {
                    return params.DID_BUILD_API == "true"
                }
            }

            steps {
                dir('department-api') {
                    sh 'pwd'
                    sh 'ls -al'
                    echo "Received Docker Image Version : ${params.DOCKER_IMAGE_VERSION}"
                    sh "sed -i 's|junsang704/department-service:.*|junsang704/department-service:${params.DOCKER_IMAGE_VERSION}|g' rollout.yaml"
                    sh 'cat rollout.yaml'
                }
            }
        }

        stage('Commit & Push') {
            when {
                expression { 
                    return params.DID_BUILD_API == "true" ||  params.DID_BUILD_APP == "true"
                }
            }

            steps {
                sh 'git config --list'
                sh 'git config user.name "jenkins"'
                sh 'git config user.email "jenkins@beyond.com"'
                sh 'git config --list'
                sh "git add ."
                sh "git commit -m 'Update Image Version ${params.DOCKER_IMAGE_VERSION}'"
                sh 'git status'

                sshagent(['github-k8s-manifests']) {
                    sh 'git push'
                }
            }
        }
    }