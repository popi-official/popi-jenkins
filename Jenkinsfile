pipeline {
    agent any

    environment {
        GITHUB_REPO = 'https://github.com/popi-official/popi-user-server.git'
        ECR_REPO = '446305434496.dkr.ecr.ap-northeast-2.amazonaws.com/popi-user'
        ECR_CREDENTIALS_ID = 'ecr:ap-northeast-2:aws-ecr'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'develop', url: "${GITHUB_REPO}"
                sh 'git fetch origin develop'
            }
        }

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

                                slackSend (channel: '#popi-jenkins-user', color: '#00FF00', message: "ECR push 성공: ${module} [빌드 번호: ${imageTag}]")

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
    }
}