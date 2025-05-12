## Clone Repository
```groovy
stage(':::::::::: CLONE REPOSITORY ::::::::::') {
    steps {
        script{
            sh """
                git config --global http.sslVerify false
                git clone -b '${branch}' --single-branch ${protocol}://${username}:${token}@${repoUri}
               """
        }
    }
}
```

## Script Build
```groovy
stage(':::::::::: SCRIPT BUILD ::::::::::') {
    steps {
        script {
            docker.withRegistry('', 'tps-docker') {
                docker.image("${buildDockerImage}").inside("--network host --privileged -u 0") {
                    sh """${buildScript}"""
                }
            }
        }
    }
}
```

## Docker file in Source Build / Image Push
```groovy
stage(':::::::::: DOCKER FILE IN SOURCE BUILD / IMAGE PUSH ::::::::::') {
    steps {
        script {
            dir(workingDir) {
                docker.withRegistry('', 'tps-docker') {
                    sh """docker run -e DOCKER_CONFIG=/.docker -v /root/.docker:/.docker -v ./:/workspace gcr.io/kaniko-project/executor:v1.21.0-debug --dockerfile "./${dockerfileDirPath}${dockerfileName}" --destination "${host}/${project}/${module}:${imageTag}" --context dir:///workspace/ --insecure --skip-tls-verify"""
                }
            }
        }
    }
}
```

## Docker file in Script / Image Push
```groovy
stage(':::::::::: DOCKER FILE IN SCRIPT / IMAGE PUSH ::::::::::') {
    steps {
        script {
            wrap([$class: "MaskPasswordsBuildWrapper", varPasswordPairs: [[password: globalDockerRepositoryPassword]]]) {
                sh """
                   docker login ${host}
                   """
                }
                sh """
                  echo "${dockerFile}" > ./Dockerfile
                  cat ./Dockerfile
                  """
                sh """
                  docker build -t ${image} .
                  docker images
                   """
                sh """
                  docker push --all-tags ${image}
                   """
                sh """
                  docker rmi ${image}
                  """
        }
    }
}
```


## Connect Remote
```groovy
stage(':::::::::: CONNECT REMOTE ::::::::::') {
  steps {
      script {
          def remote = [:]
          remote.name = ""
          remote.host = ""
          remote.port = ""

          withCredentials([usernamePassword(credentialsId: '', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]){
              remote.user = USERNAME
              remote.password = PASSWORD
              remote.allowAnyHosts = true
          }
      }
   }
}
```


## Remote File Transfer
```groovy
stage(':::::::::: REMOTE FILE TRANSFER ::::::::::') {
    steps {
        script {
            sshCommand remote: remote, command: ''
            sshPut remote: remote, from: '', into: ''
        }
    }
}
```


## Deploy Kubernetes by Script
```groovy
stage(':::::::::: DEPLOY KUBERNETES BY SCRIPT ::::::::::') {
    steps {
        script {
            sh '''echo "${containerConfig}" > deploy.yaml'''
            sh 'cat deploy.yaml'

            withKubeConfig([credentialsId: '${credentialId}', serverUrl: '${credentialUrl}']) {
                sh """kubectl apply -f deploy.yaml"""
                sh """kubectl rollout restart deploy ${deploymentName} -n ${namespace}"""
            }
        }
    }
}
```


## Deploy Kubernetes by ArgoCD
```groovy
stage(':::::::::: DEPLOY KUBERNETES BY ArgoCD ::::::::::') {
    steps {
        script {
            sh """
            git init
            git remote add origin ${gitURL}
            git remote set-url origin ${gitURLwithCredential}
            git pull origin ${branch}
            git checkout ${branch}
            git add ${file}
            git commit -m "${message}"
            git push origin ${branch}
        """
        }
    }
}
```


## Sample pipeline
```groovy
pipeline {
	agent any
	environment {
		branch = "dev"
		protocol = "http"
		repoUri = "demo.gitlab.com/sample/demo-1.git"
		manifestRepoUri = "demo.gitlab.com/system/manifest.git"
		imageRegistry = "https://demo.harbor.com:8443"
		GIT_TOKEN = "glpat-djald3LdakTfhXd"
		APP_NAME = "demo-1"
		IMAGE_NAME = "demo.harbor.com:8443/demo/demo-1"
		BASE_IMAGE = "demo.harbor.com:8443/library/demobase:latest"
		MANIFEST_PATH = "SAMPLE/demo-1/dev/SAMPLE-demo-1-dev-deployment.yaml"
	}

	tools {
		maven "maven-3.6.0"
	}

	stages {
		stage(':::::::: CLEAN ::::::::') {
			steps {
				deleteDir()
			}
		}

		stage(':::::::: CHECKOUT ::::::::') {
			steps {
				script {
					dir('app') {
						checkout([
							$class: 'GitSCM',
							branches: [[name: "${branch}"]],
							userRemoteConfigs: [[url: "${protocol}://oauth2:${GIT_TOKEN}@${repoUri}"]]
						])
						sh '''
						echo "$(git rev-parse --short HEAD)T$(date +%s)" > ./tag_name
						'''
					}
				}
			}
		}

		stage(':::::::: MAVEN BUILD ::::::::') {
			steps {
				script {
					dir('app') {
						sh '''
						mvn clean package \
						 -U -X \
						 -Dmaven.repo.local=/share/repostory \
						 -Dmaven.test.skip=true
						echo $(ls target/*.jar | head -n 1) > ./jar_name
						'''
					}
				}
			}
		}

		stage(':::::::: IMAGE BUILD ::::::::') {
			steps {
				script {
					dir('app') {
						def jar_name = sh(script: "cat ./jar_name", returnStdout: true).trim()
						def tag_name = sh(script: "cat ./tag_name", returnStdout: true).trim()
						writeFile file: 'Dockerfile', text: """
FROM ${BASE_IMAGE}
...
						"""
						sh 'cat ./Dockerfile'

						docker.withRegistry(image_registry, "harbor-credential-name") {
							def image = docker.build("${IMAGE_NAME}:${tag_name}")
							image.push()
						}

						sh "docker rmi ${IMAGE_NAME}:${tag_name}
					}
				}
			}
		}

		stage(':::::::: DEPLOY ::::::::') {
			steps {
				script {
					dir('manifest') {
						def to_tag = sh(script: "cat ../app/tag_name", returnStdout: true).trim()

						build job: 'COMMON/update-manifest',
							  wait: true,
							  parameters: [
								  string(name: 'MANIFEST_PATH', value: env.MANIFEST_PATH),
								  string(name: 'IMAGE_NAME', value: env.IMAGE_NAME),
								  string(name: 'TO_TAG', value: to_tag),
								  string(name: 'APP_NAME', value: env.APP_NAME),
								  string(name: 'ENV', value: env.branch)
							  ]
					}
				}
			}
		}
	}
}
```

# 수정해야함
# ================================================================================
```groovy
pipeline {
  agent { 
    label 'dind'
  }
  environment {
      DEPLOYMENT_NAME = 'app-be'
      GITOPS_URL = "http://gitea-http:3000/git-ops/app-test-be.git"
      GITOPS_PATH = 'gitea-http:3000/git-ops/app-test-be.git'
      GITOPS_CREDENTIALS_ID = 'gitea-jenkins-token'
      GITOPS_BRANCH = 'main'
      GITOPS_DIR = 'dev'
      DOCKER_IMAGE_URL = 'cicd.harbor.dev.eris.go.kr/test-app/back'
      CONTAINER_NAME = 'app-be'
  }
  stages {
    stage('Install kubectl') {
      steps {
        sh '''
          curl -LO "https://dl.k8s.io/release/v1.27.10/bin/linux/amd64/kubectl"
          chmod +x kubectl
          mkdir -p $HOME/bin
          mv kubectl $HOME/bin/
        '''
      }
    }
    stage('Kubernetes Rollback') {
      steps {
        withEnv(["PATH+EXTRA=$HOME/bin"]) {
          withKubeConfig([
            credentialsId: 'kubeconfig-test',
            namespace: 'test-dev',
            serverUrl: 'https://kubernetes.default'
          ]) {
              script {
                 def beforeRollback = sh(
                    script: "kubectl get deployment $DEPLOYMENT_NAME -o jsonpath='{..image}'",
                    returnStdout: true
                ).trim()
                echo "Before Rollback Version : ${beforeRollback}"

                sh "kubectl rollout undo deployment/$DEPLOYMENT_NAME"
                def afterRollback = sh(
                    script: "kubectl get deployment $DEPLOYMENT_NAME -o jsonpath='{..image}'",
                    returnStdout: true
                ).trim()
                echo "After Rollback Version : ${afterRollback}"
                env.AFTER_ROLLBACK_VERSION = afterRollback
              }
          }
        }
      }
    }
    stage('Update GitOps') {
        steps {
                git (
                    url: "http://${GITOPS_PATH}",
                    branch: GITOPS_BRANCH,
                    credentialsId: GITOPS_CREDENTIALS_ID
                )

                script {
                    dir(GITOPS_DIR){
                        def currentVersion = sh(script: "grep 'image:' deployment.yaml | awk -F ':' '{print \$NF}'", returnStdout: true).trim()
                        echo "현재 버전: ${currentVersion}"
                        echo "롤백된 버전: ${env.AFTER_ROLLBACK_VERSION}"
                        
                        if (currentVersion != env.AFTER_ROLLBACK_VERSION) {
                            echo "버전이 다릅니다. 업데이트를 진행합니다."
                            sh """
                                sed -i "/name: ${CONTAINER_NAME}/{n;s|^\\s*image: [^ ]*|  image: ${DOCKER_IMAGE_URL}:${env.AFTER_ROLLBACK_VERSION}|}" deployment.yaml
                            """
            
                            sh 'git config --global user.email "jenkins@okestro.com"'
                            sh 'git config --global user.name "jenkins"'
                            sh 'git add deployment.yaml'
                            sh "git commit -m 'Update image version: ${env.AFTER_ROLLBACK_VERSION}'"
                            withCredentials([usernamePassword(credentialsId: GITOPS_CREDENTIALS_ID, passwordVariable: 'GITOPS_PASSWORD', usernameVariable: 'GITOPS_USERNAME')]) {
                                sh "git push http://${GITOPS_USERNAME}:${GITOPS_PASSWORD}@${GITOPS_PATH} ${GITOPS_BRANCH}"
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

###

```groovy
pipeline {
    parameters {
        gitParameter(
            name: 'SELECTED_TAG',
            description: '배포할 Git 태그를 선택하세요',
            type: 'PT_TAG',
            branch: 'master',
            branchFilter: 'origin/(.*)',
            defaultValue: 'latest',
            useRepository: 'http://gitea-http:3000/test-app/test-be.git',
            sortMode: 'DESCENDING_SMART'
        )
    }
    agent {
        kubernetes {
            label 'dind'
        }
    }
    
    // 파이프라인 단계 정의
    environment {
        GIT_URL = 'http://gitea-http:3000/test-app/test-be.git'
        GIT_CREDENTIALS_ID = 'gitea-jenkins-token'
        GIT_BRANCH = 'master'
        GITOPS_URL = 'http://gitea-http:3000/git-ops/app-test-be.git'
        GITOPS_PATH = 'gitea-http:3000/git-ops/app-test-be.git'
        GITOPS_CREDENTIALS_ID = 'gitea-jenkins-token'
        GITOPS_BRANCH = 'main'
        GITOPS_DIR = 'dev'
        DOCKER_IMAGE_URL = 'cicd.harbor.dev.eris.go.kr/test-app/back'
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "refs/tags/${params.SELECTED_TAG}"]],
                        extensions: [],
                        userRemoteConfigs: [
                            [
                                url: GIT_URL,
                                credentialsId: GIT_CREDENTIALS_ID,
                                refspec: '+refs/tags/*:refs/remotes/origin/tags/*'
                            ]
                        ]
                    ])
                }
                script {
                    env.APP_VERSION = params.SELECTED_TAG ?: sh(
                        script: 'git describe --tags',
                        returnStdout: true
                    ).trim()
                    echo "App Version(TAG): ${env.APP_VERSION}"
                }
            }
        }
        // Git Manifest commit
        stage('Git Manifest Push') {
            steps {
                git (
                    url: GITOPS_URL,
                    branch: GITOPS_BRANCH,
                    credentialsId: GITOPS_CREDENTIALS_ID
                )

                script {
                    dir(GITOPS_DIR){
                        def currentVersion = sh(script: "grep 'image:' deployment.yaml | awk -F ':' '{print \$NF}'", returnStdout: true).trim()
                        echo "현재 버전: ${currentVersion}"
                        echo "롤백 버전: ${APP_VERSION}"
                        
                        if (currentVersion != APP_VERSION) {
                            echo "롤백을 실행합니다."
                            sh """
                                sed -i "/name: app-be/{n;s|image: .*|image: ${DOCKER_IMAGE_URL}:${APP_VERSION}|}" deployment.yaml
                            """

                            sh 'git config --global user.email "jenkins@okestro.com"'
                            sh 'git config --global user.name "jenkins"'
                            sh 'git add deployment.yaml'
                            sh 'git commit -m "Update image version: ${APP_VERSION}"'
                            sh "git tag ${APP_VERSION}"
                            withCredentials([usernamePassword(credentialsId: GITOPS_CREDENTIALS_ID, passwordVariable: 'GITOPS_PASSWORD', usernameVariable: 'GITOPS_USERNAME')]) {
                                sh 'git push http://${GITOPS_USERNAME}:${GITOPS_PASSWORD}@${GITOPS_PATH} ${GITOPS_BRANCH}'
                                sh 'git push http://${GITOPS_USERNAME}:${GITOPS_PASSWORD}@${GITOPS_PATH} ${APP_VERSION}'
                            }
                        } else {
                            echo "버전이 동일합니다. 롤백이 실행되지 않습니다."
                        }
                    }
                }
            }   
        }
    }
}
```

```groovy
pipeline {
    parameters {
        gitParameter(
            name: 'SELECTED_TAG',
            description: '배포할 Git 태그를 선택하세요',
            type: 'PT_TAG',
            branch: 'master',
            branchFilter: 'origin/(.*)',
            defaultValue: 'latest',
            useRepository: 'http://gitea-http:3000/test-app/test-be.git',
            sortMode: 'DESCENDING_SMART'
        )
    }
    agent {
        kubernetes {
            label 'dind'
        }
    }
    tools {
        jdk 'jdk-17'
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
        GITOPS_CONTAINER_NAME = 'app-be'
        DOCKER_REGISTRY_URL = 'harbor-harbor-core:80'
        DOCKER_PROJECT_NAME = 'test-app'
        DOCKER_IMAGE_NAME = 'test-app/back'
        DOCKER_REPO_NAME = 'back'
        DOCKER_IMAGE_URL = 'cicd.harbor.dev.eris.go.kr/test-app/back'
        DOCKER_REGISTRY_CREDENTIALS_ID = 'test-harbor-robot'
        HARBOR_SERVER = 'Harbor'
    }

    stages {
        stage('Git Checkout') {
            steps {
                script {
                    try {
                        echo "===============[Git Checkout] 단계 시작==============="

                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: "refs/tags/${params.SELECTED_TAG}"]],
                            extensions: [],
                            userRemoteConfigs: [
                                [
                                    url: GIT_URL,
                                    credentialsId: GIT_CREDENTIALS_ID,
                                    refspec: '+refs/tags/*:refs/remotes/origin/tags/*'
                                ]
                            ]
                        ])

                        env.APP_VERSION = params.SELECTED_TAG ?: sh(
                            script: 'git describe --tags',
                            returnStdout: true
                        ).trim()
                        echo "App Version(TAG): ${env.APP_VERSION}"

                        env.PREVIOUS_VERSION = sh(
                            script: "git tag --sort=-v:refname | grep -A 1 "${params.SELECTED_TAG}" | tail -n 1",
                            returnStdout: true
                        ).trim()
                        echo "App Version(PREV): ${env.PREVIOUS_VERSION}"

                        echo "===============[Git Checkout] 단계 종료==============="
                    } catch (err) {
                        echo "[Git Checkout] 실패: ${err}"
                        error("Git 체크아웃 단계에서 오류 발생")
                    }
                }
            }
        }
        stage('Gradle Build') {
			steps {
                echo "===============[Gradle Build] 단계 시작==============="

			    script {
                    def lastError = null
                    retry(3) {
                        try {
                            sh 'chmod +x ./gradlew'
                            sh './gradlew clean build'
                        } catch (err) {
                            lastError = err
                            
                            if (err.getMessage().contains('not found') || err.getMessage().contains('Permission denied')) {
                                echo "Gradle 실행 자체 실패 (Gradle down)"
                            } else {
                                echo "Gradle 빌드 실패 (컴파일/테스트/빌드 오류)"
                            }
                            throw err
                        }
                    }
                }

                echo "===============[Gradle Build] 단계 종료==============="
            }
		}
        stage('Docker Build') {
            steps {
                echo "===============[Docker Build] 단계 시작==============="
                script {
                    try {
                        container('docker-dind') {
                            sh """
                                docker build -t ${DOCKER_REGISTRY_URL}/${DOCKER_IMAGE_NAME}:${APP_VERSION} .
                                docker tag ${DOCKER_REGISTRY_URL}/${DOCKER_IMAGE_NAME}:${APP_VERSION} ${DOCKER_REGISTRY_URL}/${DOCKER_IMAGE_NAME}:latest
                            """
                        }
                    } catch (err) {
                        echo "Docker Build 실패: ${err.getMessage()}"
                        error("Docker Build 단계에서 오류 발생")
                    }
                }
                echo "===============[Docker Build] 단계 종료==============="
            }
        }
        stage('Docker Push') {
            steps {
                echo "===============[Docker Push] 단계 시작==============="

                container('docker-dind') {
                    withCredentials([usernamePassword(
                        credentialsId: 'test-harbor-robot',
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASS'
                    )]) {
                        sh """
                            echo '$HARBOR_PASS' | docker login "${DOCKER_REGISTRY_URL}" -u '$HARBOR_USER' --password-stdin
                            docker push ${DOCKER_REGISTRY_URL}/${DOCKER_IMAGE_NAME}:${APP_VERSION}
                        """   
                    }
                }

                echo "===============[Docker Push] 단계 종료==============="
            }
        }
        stage('Harbor Image Scan') {
            steps {
                echo "===============[Harbor Image Scan] 단계 시작==============="

                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'harbor-admin',
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASS'
                    )]) {
                        sh """
                            curl -v -k -X POST \\
                              -u "\$HARBOR_USER:\$HARBOR_PASS" \\
                              "http://${DOCKER_REGISTRY_URL}/api/v2.0/projects/${DOCKER_PROJECT_NAME}/repositories/${DOCKER_REPO_NAME}/artifacts/${APP_VERSION}/scan"
                        """
                        
                        def maxWait = 150   // 최대 2분 30초 대기
                        def interval = 10   // 10초마다 재시도
                        def waited = 0
                        def vulnResult = '{}'
                        def found = false
        
                        while (waited < maxWait) {
                            vulnResult = sh(
                                script: """
                                    curl -sk -u "$HARBOR_USER:$HARBOR_PASS" \
                                    "http://${DOCKER_REGISTRY_URL}/api/v2.0/projects/${DOCKER_PROJECT_NAME}/repositories/${DOCKER_REPO_NAME}/artifacts/${APP_VERSION}/additions/vulnerabilities"
                                """,
                                returnStdout: true
                            ).trim()
                            
                            if (vulnResult != '{}' && vulnResult != '[]' && vulnResult.length() > 5) {
                                found = true
                                break
                            }
                            echo "아직 취약점 데이터가 없습니다. ${interval}초 후 재시도 (경과: ${waited}s)"
                            sleep interval
                            waited += interval
                        }
        
                        if (!found) {
                            error "Harbor 취약점 결과를 ${maxWait}초 내에 받지 못했습니다."
                        }

                        // 3. JSON 파싱 및 등급/코드 추출
                        def parsed = readJSON text: vulnResult
                        def reportKey = parsed.keySet().find { it.startsWith('application/vnd.scanner.adapter.vuln.report.harbor') }
                        def report = parsed[reportKey]
                        def severity = report?.severity ?: 'Unknown'
                        def vulnerabilities = report?.vulnerabilities ?: []

                        // 등급/코드 콘솔 출력
                        echo "취약점 등급 : ${severity}"
                        def cveList = vulnerabilities.collect { it.id }.join(', ')
                        echo "취약점 코드 : ${cveList ?: '없음'}"

                        // 4. 등급에 따라 분기
                        if (severity.toLowerCase() == 'critical') {
                            error("Critical 취약점 발견: 파이프라인 실패 처리")
                        } else {
                            echo "Critical 취약점 없음, docker push latest 실행"
                            container('docker-dind') {
                                withCredentials([usernamePassword(
                                    credentialsId: 'test-harbor-robot',
                                    usernameVariable: 'HARBOR_USER',
                                    passwordVariable: 'HARBOR_PASS'
                                )]) {
                                    sh "docker push ${DOCKER_REGISTRY_URL}/${DOCKER_IMAGE_NAME}:latest"
                                }
                            }
                        }

                        echo "취약점 상세 결과: ${vulnResult}"
                    }
                }
                echo "===============[Harbor Image Scan] 단계 종료==============="
            }
        }
        stage('GitOps Deploy') {
            steps {
                echo "===============[GitOps Deploy] 단계 시작==============="

                git (
                    url: GITOPS_URL,
                    branch: GITOPS_BRANCH,
                    credentialsId: GITOPS_CREDENTIALS_ID
                )

                script {
                    dir(GITOPS_DIR){
                        try {
                            def currentVersion = sh(script: "grep 'image:' deployment.yaml | awk -F ':' '{print \$NF}' | head -n 1", returnStdout: true).trim()
                            echo "현재 버전: ${currentVersion}"
                            echo "신규 버전: ${APP_VERSION}"
                            
                            if (currentVersion != APP_VERSION) {
                                echo "버전이 다릅니다. 업데이트를 진행합니다."
                                sh """
                                    sed -i "/name: ${GITOPS_CONTAINER_NAME}/{n;s|image: .*|image: ${DOCKER_IMAGE_URL}:${APP_VERSION}|}" deployment.yaml
                                """

                                sh 'git config --global user.email "jenkins@okestro.com"'
                                sh 'git config --global user.name "jenkins"'
                                sh 'git add deployment.yaml'
                                sh 'git commit -m "Update image version: ${APP_VERSION}"'
                                withCredentials([usernamePassword(credentialsId: GITOPS_CREDENTIALS_ID, passwordVariable: 'GITOPS_PASSWORD', usernameVariable: 'GITOPS_USERNAME')]) {
                                    sh 'git push http://${GITOPS_USERNAME}:${GITOPS_PASSWORD}@${GITOPS_PATH} ${GITOPS_BRANCH}'
                                }
                            } else {
                                echo "버전이 동일합니다. 업데이트가 필요하지 않습니다."
                            }
                        } catch (err) {
                            // 2. 실패 시 롤백 수행
                            echo "[롤백] GitOps Deploy 단계에서 오류 발생! deployment.yaml을 이전 상태로 복원합니다."
                            sh 'git checkout HEAD~1 deployment.yaml'
                            sh 'git add deployment.yaml'
                            sh 'git commit -m "[Rollback]: 배포오류로 인한 롤백실행"'
                            withCredentials([usernamePassword(credentialsId: GITOPS_CREDENTIALS_ID, passwordVariable: 'GITOPS_PASSWORD', usernameVariable: 'GITOPS_USERNAME')]) {
                                sh 'git push http://${GITOPS_USERNAME}:${GITOPS_PASSWORD}@${GITOPS_PATH} ${GITOPS_BRANCH}'
                            }
                            error "GitOps Deploy 실패 및 롤백 완료: ${err}"
                        }
                    }
                }

                echo "===============[GitOps Deploy] 단계 종료==============="
            }   
        }
    }
}
```


pipeline {
    parameters {
        gitParameter(
            name: 'SELECTED_TAG',
            description: '배포할 Git 태그를 선택하세요',
            type: 'PT_TAG',
            branch: 'master',
            branchFilter: 'origin/(.*)',
            defaultValue: 'latest',
            useRepository: 'http://gitea-http:3000/test-app/test-be.git',
            sortMode: 'DESCENDING_SMART'
        )
    }
    agent {
        kubernetes {
            label 'dind'
        }
    }
    tools {
        jdk 'jdk-17'
    }
    
    // 파이프라인 단계 정의
    environment {
        GIT_URL = 'http://gitea-http:3000/test-app/test-be.git'
        GIT_CREDENTIALS_ID = 'gitea-jenkins-token'
        GIT_BRANCH = 'master'
        GITOPS_URL = 'http://gitea-http:3000/git-ops/app-test-be.git'
        GITOPS_PATH = 'gitea-http:3000/git-ops/app-test-be.git'
        GITOPS_CREDENTIALS_ID = 'gitea-jenkins-token'
        GITOPS_BRANCH = 'main'
        GITOPS_DIR = 'dev'
        GITOPS_CONTAINER_NAME = 'app-be'
        DOCKER_REGISTRY_URL = 'cicd.harbor.dev.eris.go.kr'
        DOCKER_PROJECT_NAME = 'test-app'
        DOCKER_IMAGE_NAME = 'test-app/back'
        DOCKER_REPO_NAME = 'back'
        DOCKER_IMAGE_URL = 'cicd.harbor.dev.eris.go.kr/test-app/back'
        DOCKER_REGISTRY_CREDENTIALS_ID = 'test-harbor-robot'
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "refs/tags/${params.SELECTED_TAG}"]],
                        extensions: [],
                        userRemoteConfigs: [
                            [
                                url: GIT_URL,
                                credentialsId: GIT_CREDENTIALS_ID,
                                refspec: '+refs/tags/*:refs/remotes/origin/tags/*'
                            ]
                        ]
                    ])
                }
                script {
                    env.APP_VERSION = params.SELECTED_TAG ?: sh(
                        script: 'git describe --tags',
                        returnStdout: true
                    ).trim()
                    echo "App Version(TAG): ${env.APP_VERSION}"
                }
            }
        }
        stage('Gradle Build') {
			steps {
			    retry(3) {
		            sh "chmod +x ./gradlew"
			        sh "./gradlew clean build"       
			    }
			}
		}
        stage('Docker Build') {
            steps {
                container('docker-dind') {
                    sh """
                        docker build -t harbor-harbor-core:80/${DOCKER_IMAGE_NAME}:${APP_VERSION} .
                    """   
                }
            }
        }
        stage('Docker Push') {
            steps {
                container('docker-dind') {
                    withCredentials([usernamePassword(
                        credentialsId: 'test-harbor-robot',
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASS'
                    )]) {
                        sh """
                            echo '$HARBOR_PASS' | docker login 'harbor-harbor-core:80' -u '$HARBOR_USER' --password-stdin
                            docker push harbor-harbor-core:80/${DOCKER_IMAGE_NAME}:${APP_VERSION}
                        """   
                    }
                }
            }
        }
        stage('Harbor Scan Trigger') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'test-harbor-robot',
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASS'
                    )]) {
                        sh """
                            curl -v -k -X POST \\
                              -u "\$HARBOR_USER:\$HARBOR_PASS" \\
                              "harbor-harbor-core:80/api/v2.0/projects/${DOCKER_PROJECT_NAME}/repositories/${DOCKER_REPO_NAME}/artifacts/${APP_VERSION}/scan"
                        """
                    }
                }
            }
        }
        // Git Manifest commit
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
                        echo "신규 버전: ${APP_VERSION}"
                        
                        if (currentVersion != APP_VERSION) {
                            echo "버전이 다릅니다. 업데이트를 진행합니다."
                            sh """
                                sed -i "/name: app-be/{n;s|image: .*|image: ${DOCKER_IMAGE_URL}:${APP_VERSION}|}" deployment.yaml
                            """

                            sh 'git config --global user.email "jenkins@okestro.com"'
                            sh 'git config --global user.name "jenkins"'
                            sh 'git add deployment.yaml'
                            sh 'git commit -m "Update image version: ${APP_VERSION}"'
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





