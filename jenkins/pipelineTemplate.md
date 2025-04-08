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