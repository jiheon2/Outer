```groovy
pipeline {
    agent {
        kubernetes {
            label 'jenkins-agent'
        }
    }
    
    environment {
        GIT_URL = 'http://gitea-http:3000/test-app/test-be.git'
        GIT_CREDENTIALS_ID = 'gitea-jenkins-token'
        GIT_BRANCH = 'master'
        GITOPS_URL = 'http://gitea-http:3000/git-ops/app-test-be.git'
        GITOPS_PATH = 'gitea-http:3000/git-ops/app-test-be.git'
        GITOPS_CREDENTIALS_ID = 'gitea-jenkins-token'
        GITOPS_BRANCH = 'main'
        GITOPS_DIR = 'dev'
        DOCKER_IMAGE_URL  = 'cicd.harbor.dev.eris.go.kr/test-app/back'
        GITOPS_CONTAINER_NAME = 'app-be'
    }
    
    stages {
        stage('Git Checkout') {
            steps {
                git (
                    url: GIT_URL,
                    branch: GIT_BRANCH,
                    credentialsId: GIT_CREDENTIALS_ID
                )   
            }
        }
        stage('GitOps Rollback') {
            steps {
                echo "===============[GitOps Rollback] 단계 시작==============="

                git (
                    url: GITOPS_URL,
                    branch: GITOPS_BRANCH,
                    credentialsId: GITOPS_CREDENTIALS_ID
                )

                script {
                    dir(GITOPS_DIR) {
                        withCredentials([
                            usernamePassword(
                                credentialsId: GITOPS_CREDENTIALS_ID, 
                                passwordVariable: 'GITOPS_PASSWORD', 
                                usernameVariable: 'GITOPS_USERNAME'
                            )
                        ]) {
                            try {
                                def currentVersion = sh(script: "grep 'image:' deployment.yaml | awk -F ':' '{print \$NF}' | head -n 1", returnStdout: true).trim()
                                echo "현재 버전: ${currentVersion}"
                                echo "롤백 버전: ${env.TAGS}"
                                                        
                                echo "롤백을 실행합니다."
                                sh """
                                    sed -i "/name: ${GITOPS_CONTAINER_NAME}/{n;s|image: .*|image: ${DOCKER_IMAGE_URL}:${env.TAGS}|}" deployment.yaml
                                    git config --global user.email "jenkins@okestro.com"
                                    git config --global user.name "jenkins"
                                    git add deployment.yaml
                                    git commit -m "Update image version: ${env.TAGS}"
                                    git push http://${GITOPS_USERNAME}:${GITOPS_PASSWORD}@${GITOPS_PATH} ${GITOPS_BRANCH}
                                """
                                echo "롤백을 완료하였습니다."

                            } catch (err) {
                                // 2. 실패 시 롤백 수행
                                echo "[롤백] GitOps Rollback 단계에서 오류 발생! deployment.yaml을 이전 상태로 복원합니다."
                                sh """
                                    git checkout HEAD~1 deployment.yaml
                                    git add deployment.yaml
                                    git commit -m "[Rollback]: 배포오류로 인한 롤백실행"
                                    git push http://${GITOPS_USERNAME}:${GITOPS_PASSWORD}@${GITOPS_PATH} ${GITOPS_BRANCH}
                                """ 
                                error "GitOps Rollback 실패 및 상태 복구 완료: ${err}"
                            }
                        } 
                    }
                }
                echo "===============[GitOps Rollback] 단계 종료==============="
            }   
        }
    }
}
```


```groovy
pipeline {
    agent {
        kubernetes {
            label 'jenkins-agent'
        }
    }
    
    environment {
        GIT_URL = 'http://gitea-http:3000/test-app/test-be.git'
        GIT_CREDENTIALS_ID = 'gitea-jenkins-token'
        GIT_BRANCH = 'master'
        GITOPS_URL = 'http://gitea-http:3000/git-ops/app-test-be.git'
        GITOPS_PATH = 'gitea-http:3000/git-ops/app-test-be.git'
        GITOPS_CREDENTIALS_ID = 'gitea-jenkins-token'
        GITOPS_BRANCH = 'main'
        GITOPS_DIR = 'dev'
        DOCKER_IMAGE_URL  = 'cicd.harbor.dev.eris.go.kr/test-app/back'
    }
    
    stages {
        stage('Git Checkout') {
            steps {
                git (
                    url: GIT_URL,
                    branch: GIT_BRANCH,
                    credentialsId: GIT_CREDENTIALS_ID
                )   
            }
        }
        stage('Git Manifest Push') {
            steps {
                git (
                    url: GITOPS_URL,
                    branch: GITOPS_BRANCH,
                    credentialsId: GITOPS_CREDENTIALS_ID
                )

                script {
                    dir(GITOPS_DIR){
                        def currentVersion = sh(script: "grep 'image:' deployment.yaml | awk -F ':' '{print \$NF}' | head -n 1", returnStdout: true).trim()
                        echo "현재 버전: ${currentVersion}"
                        echo "선택 버전: ${env.TAGS}"
                        
                        if (currentVersion != env.TAGS) {
                            echo "버전이 다릅니다. 업데이트를 진행합니다."
                            sh """
                                sed -i "/name: app-be/{n;s|image: .*|image: ${DOCKER_IMAGE_URL}:${env.TAGS}|}" deployment.yaml
                            """
                            sh 'git config --global user.email "jenkins@okestro.com"'
                            sh 'git config --global user.name "jenkins"'
                            sh 'git add deployment.yaml'
                            sh "git commit -m 'Update image version: ${env.TAGS}'"
                            withCredentials([usernamePassword(credentialsId: GITOPS_CREDENTIALS_ID, passwordVariable: 'GITOPS_PASSWORD', usernameVariable: 'GITOPS_USERNAME')]) {
                                sh 'git push http://${GITOPS_USERNAME}:${GITOPS_PASSWORD}@${GITOPS_PATH} ${GITOPS_BRANCH}'
                            }
                        } else {
                            echo "버전이 동일합니다. 업데이트가 필요하지 않습니다."
                        }
                    }
                }
            }   
        }
    }
}
```