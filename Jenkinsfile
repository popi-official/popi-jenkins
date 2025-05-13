pipeline {
    agent any

    environment {
        SOURCE_REPO = 'https://github.com/popi-official/popi-user-server.git'
        ECR_REPO = '446305434496.dkr.ecr.ap-northeast-2.amazonaws.com/popi-user'
        ECR_CREDENTIALS_ID = 'ecr:ap-northeast-2:aws-ecr'
        GITOPS_REPO = 'https://github.com/popi-official/popi-ops.git'
        BASE_DIRECTORY = ''
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'develop', url: "${SOURCE_REPO}"
                sh 'git fetch origin develop'
            }
        }

        def updateModules = []

        stage('Build & Push Changed Modules') {
            steps {
                script {
                    def module_list = ['popi-api-gateway', 'popi-auth-service']

                    // origin/develop과 비교하여 변경된 파일 목록
                    def changedFiles = sh(script: "git diff --name-only HEAD~1 HEAD", returnStdout: true).trim().split("\n")

                    echo "Changed files since origin/develop:\n${changedFiles.join('\n')}"

                    module_list.each { module ->
                        def isModuleChanged = changedFiles.any {
                            it.startsWith("${module}/") ||
                            it == "build.gradle" ||
                            it == "settings.gradle"
                        }

                        if (isModuleChanged) {
                            echo "변경된 모듈 감지됨: ${module} — 빌드 및 배포 수행"

                            try {
                                // 1. 빌드
                                sh "./gradlew clean :${module}:build -x test"
                                def imageTag = "${env.BUILD_NUMBER}"

                                dir(module) {
                                    // 2. Dockerfile 생성
                                    def dockerfileContent = """
                                    FROM openjdk:17-slim
                                    WORKDIR /popi
                                    COPY ./build/libs/${module}-0.0.1-SNAPSHOT.jar app.jar
                                    ENTRYPOINT ["java", "-jar", "app.jar"]
                                    """.stripIndent()
                                    writeFile file: "Dockerfile", text: dockerfileContent

                                    // 3. Docker 이미지 빌드
                                    def imageName = "${ECR_REPO}/${module}"
                                    def image = docker.build("${imageName}:${imageTag}", ".")

                                    // 4. ECR 푸시
                                    docker.withRegistry("https://${ECR_REPO}", "${ECR_CREDENTIALS_ID}") {
                                        image.push(imageTag)
                                    }
                                }

                                // 5. slack 알림 전송
                                slackSend (channel: '#popi-jenkins-user', color: '#00FF00', message: "ECR push 성공: ${module} [빌드 번호: ${imageTag}]")

                                // 6. 변경된 모듈 저장
                                updateModules << [name: module, tag: imageTag]

                            } catch (e) {
                                slackSend (channel: '#popi-jenkins-user', color: '#FF0000', message: "ECR push 실패: ${module}")
                                echo "${module} 빌드 실패"
                            }

                        } else {
                            echo "${module} 변경 없음 — 스킵"
                        }
                    }
                }
            }
        }

        stage('Pull ArgoCD gitOps repository') {
            steps {
                git branch: 'main', url: "${GITOPS_REPO}"
            }
        }

        stage('Install kustomize') {
            steps {
                script {
                    def kustomizePath = "${env.WORKSPACE}/kustomize"
                    if (!fileExists(kustomizePath)) {
                        sh '''
                            curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
                        '''
                    }
                    sh "${kustomizePath} version"
                }
            }
        }

        stage('Change image tag') {
            steps {
                script {
                    def kustomizePath = "${env.WORKSPACE}/kustomize"

                    updateModules.each { module ->
                        def moduleName = module.name
                        def tag = module.tag
                        def imageName = "${ECR_REPO}/${moduleName}"
                        def targetDir = "${BASE_DIRECTORY}/${moduleName}"

                        echo "${moduleName} → GitOps image 태그 수정 (${imageName}:${tag})"

                        dir(targetDir) {
                            sh "${kustomizePath} edit set image ${imageName}:${tag}"
                        }
                    }
                }
            }
        }

        stage('Push to manifest repo') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'popi-config',
                        usernameVariable: 'gitUsername',
                        passwordVariable: 'gitPassword'
                    ),
                    string(credentialsId: 'github-email', variable: 'gitEmail')
                ]) {
                    script {
                        sh "git config user.email ${gitEmail}"
                        sh "git config user.name ${gitUsername}"
                        sh "git add -A"
                        sh "git commit -m '[jenkins] update image tag' || echo 'No changes to commit'"
                        sh "git push https://${gitUsername}:${gitPassword}@/github.com/popi-official/popi-ops.git"
                    }
                }
            }
        }
    }
}