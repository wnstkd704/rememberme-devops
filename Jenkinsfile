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
                command: [- cat]
                tty: true
                volumeMounts:
                - mountPath: /var/run/docker.sock
                  name: docker-socket
              volumes:
              - name: docker-socket
                hostPath:
                  path: /var/run/docker.sock
            '''
        }
    }

    environment {
        BACKEND_IMAGE = 'junsang704/rememberme-backend'
        FRONTEND_IMAGE = 'junsang704/rememberme-frontend'
        DOCKER_CRED_ID = 'dockerhub-access'
        // SSH Agent에 등록된 GitHub 배포 키 ID (필요시 수정)
        GITHUB_CRED_ID = 'github-k8s-manifests' 
    }

    stages {
        stage('Detect Changes') {
            steps {
                script {
                    // Git 대소문자 무시 방지 설정
                    sh 'git config core.ignorecase false'
                    
                    // 이전 커밋과 비교하여 변경된 파일 목록 추출
                    def changedFiles = sh(script: 'git diff --name-only HEAD~1', returnStdout: true).trim().split("\n")
                    echo "변경된 파일 목록:\n${changedFiles.join('\n')}"
                    
                    // 폴더 경로 기반 빌드 대상 결정
                    env.SHOULD_BUILD_APP = changedFiles.any { it.startsWith("Frontend/") } ? "true" : "false"
                    env.SHOULD_BUILD_API = changedFiles.any { it.startsWith("Backend/") } ? "true" : "false"

                    echo "프론트 빌드 여부: ${env.SHOULD_BUILD_APP}"
                    echo "백엔드 빌드 여부: ${env.SHOULD_BUILD_API}"
                }
            }
        }

        stage('Docker Build & Push') {
            when {
                expression { return env.SHOULD_BUILD_APP == "true" || env.SHOULD_BUILD_API == "true" }
            }
            steps {
                container('docker') {
                    script {
                        // 도커 로그인
                        withCredentials([usernamePassword(credentialsId: DOCKER_CRED_ID, usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                            sh "echo $PASS | docker login -u $USER --password-stdin"
                        }

                        // 프론트엔드 빌드
                        if (env.SHOULD_BUILD_APP == "true") {
                            dir('Frontend') {
                                sh "docker build -t ${env.FRONTEND_IMAGE}:${env.BUILD_NUMBER} ."
                                sh "docker push ${env.FRONTEND_IMAGE}:${env.BUILD_NUMBER}"
                            }
                        }

                        // 백엔드 빌드
                        if (env.SHOULD_BUILD_API == "true") {
                            dir('Backend') {
                                sh "docker build -t ${env.BACKEND_IMAGE}:${env.BUILD_NUMBER} ."
                                sh "docker push ${env.BACKEND_IMAGE}:${env.BUILD_NUMBER}"
                            }
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
                    // k8s 폴더 내의 배포 파일 태그 업데이트
                    if (env.SHOULD_BUILD_APP == "true") {
                        sh "sed -i 's|${env.FRONTEND_IMAGE}:.*|${env.FRONTEND_IMAGE}:${env.BUILD_NUMBER}|g' k8s/frontend/deployment.yaml"
                    }
                    if (env.SHOULD_BUILD_API == "true") {
                        sh "sed -i 's|${env.BACKEND_IMAGE}:.*|${env.BACKEND_IMAGE}:${env.BUILD_NUMBER}|g' k8s/backend/deployment.yaml"
                    }
                }
            }
        }

        stage('Push Manifests to Git') {
            when {
                expression { return env.SHOULD_BUILD_APP == "true" || env.SHOULD_BUILD_API == "true" }
            }
            steps {
                script {
                    sh 'git config user.name "jenkins"'
                    sh 'git config user.email "jenkins@beyond.com"'
                    sh 'git add k8s/'
                    sh "git commit -m 'Deploy: Update image version to ${env.BUILD_NUMBER}'"
                    
                    sshagent([env.GITHUB_CRED_ID]) {
                        sh 'git push origin main'
                    }
                }
            }
        }
    }
}