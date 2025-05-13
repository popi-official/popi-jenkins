def updateModules = []

pipeline {
    agent any

    environment {
        SOURCE_REPO = 'https://github.com/popi-official/popi-user-server.git'
        ECR_REPO = '446305434496.dkr.ecr.ap-northeast-2.amazonaws.com/popi-user'
        ECR_CREDENTIALS_ID = 'ecr:ap-northeast-2:aws-ecr'
        GITOPS_REPO = 'https://github.com/popi-official/popi-ops.git'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'develop', url: "${SOURCE_REPO}"
                sh 'git fetch origin develop'
            }
        }

        stage('Build & Push Changed Modules') {
            steps {
                script {
                    def moduleList = env.MODULE_LIST.split(',')
                    // origin/develop과 비교하여 변경된 파일 목록
                    def changedFiles = sh(script: "git diff --name-only HEAD~1 HEAD", returnStdout: true).trim().split("\n")

                    echo "Changed files since origin/develop:\n${changedFiles.join('\n')}"

                    moduleList.each { module ->
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
                                    // 2. Docker 이미지 빌드
                                    def imageName = "${ECR_REPO}/${module}"
                                    def image = docker.build("${imageName}:${imageTag}", ".")

                                    // 3. ECR 푸시
                                    docker.withRegistry("https://${ECR_REPO}", "${ECR_CREDENTIALS_ID}") {
                                        image.push(imageTag)
                                    }
                                }

                                // 4. slack 알림 전송
                                slackSend (channel: '#popi-jenkins-user', color: '#00FF00', message: "ECR push 성공: ${module} [빌드 번호: ${imageTag}]")

                                // 5. 변경된 모듈 저장
                                def updateModule = original.collect { it.replaceAll(/^popi-/, '').replaceAll(/-service$/, '') }
                                updateModules << [name: updateModule, tag: imageTag]

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

        stage('Update GitOps Repo') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'popi-gitops',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    ),
                    string(credentialsId: 'github-email', variable: 'GIT_EMAIL')
                ]) {
                    script {
                        if (updateModules && !updateModules.isEmpty()) {
                            echo "GitOps Repo를 clone합니다."

                            sh """
                            rm -rf gitops-repo
                            git config --global user.name ${GIT_USER}
                            git config --global user.email ${GIT_EMAIL}

                            git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/popi-official/popi-ops.git gitops-repo
                            """

                            updateModules.each { module ->
                                def moduleName = module.name
                                def imageTag = module.tag

                                echo "GitOps Repo의 ${moduleName} values.yaml 파일을 ${imageTag}로 업데이트합니다."

                                sh """
                                cd gitops-repo/apps/${moduleName}
                                sed -i '' "s/tag: .*/tag: ${imageTag}/" values.yaml
                                git add values.yaml
                                git commit -m "Update ${moduleName} image tag to ${imageTag}" || echo "No changes to commit"
                                cd -
                                """
                            }

                            echo "모든 변경사항을 push합니다."

                            sh """
                            cd gitops-repo
                            git push origin main
                            """
                        } else {
                            echo "updateModules가 비어 있어 GitOps 업데이트를 건너뜁니다."
                        }
                    }
                }
            }
        }
    }
}
