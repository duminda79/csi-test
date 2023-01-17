def abort					= false
def APP_DIR					= "app-delivarables"
def APP_EKS					= "kube-manifests"
def EKS_Cluster				= "test-csi"
def deployed				= false
def EKS_DEPLOY_YML			= "Deployment.yml"
def BRANCH					= ""
def DOCKER_IMAGE_REPO		= "csi"
def DOCKER_IMAGE_REPO_TAG	= "test"

pipeline {

	agent any

	// Polling changes from GIT repository every 5 minutes
	triggers {
		pollSCM "*/5 * * * *"
	}

	stages {
		stage("Check changeSets") {
			when {
				allOf {
					expression {
						env.BUILD_NUMBER != "1"
					}
					expression {
						currentBuild.changeSets.size() == 0
					}
					expression {
						currentBuild.getPreviousBuild().result == "SUCCESS"
					}
					triggeredBy cause: "BranchIndexingCause"
				}
			}

			steps {
				script {
					println "Build aborted because triggeredBy BranchIndexingCause and currentBuild.changeSets.size() == 0"

					abort = true
				}
			}
		}


        stage('CSI Packaging') {
            steps {
                sh """
                    mkdir -p ${APP_DIR}
                    cd ${APP_DIR}
					cp ../docker/Dockerfile .
					cp ../index.html .
					ls -al
                    """
            }
        }

       stage('CSI Build Docker Image'){
            steps {
			script {
					if ("$GIT_BRANCH" == "main") { // if master branch commit found then create a git tag using build number
                sh """
					echo ${BUILD_NUMBER}
					cd ${APP_DIR}
					docker build --no-cache -t 264672321272.dkr.ecr.us-east-1.amazonaws.com/${DOCKER_IMAGE_REPO}:${DOCKER_IMAGE_REPO_TAG}-v_${GIT_BRANCH}_${BUILD_NUMBER} .
                    """
					} else {		
						
						def BRANCH_NAME =  sh(script: """echo ${GIT_BRANCH}""", returnStdout: true).trim();
						String[] STORY_ID;
						
						if(BRANCH_NAME.contains('/'))
						{
					          STORY_ID  = BRANCH_NAME.split('/');
					          BRANCH = STORY_ID[1];		
						}
						else
						{
						  BRANCH = BRANCH_NAME;
						}
		   		
                sh """   
					echo ${BUILD_NUMBER}
					sed -i -e "s/\\(csi:\\).*/\\1${DOCKER_IMAGE_REPO_TAG}-v_${BRANCH}_${BUILD_NUMBER}/" ${APP_EKS}/${EKS_DEPLOY_YML}
					cat ${APP_EKS}/${EKS_DEPLOY_YML}
					cd ${APP_DIR}
					docker build --no-cache -t 264672321272.dkr.ecr.us-east-1.amazonaws.com/${DOCKER_IMAGE_REPO}:${DOCKER_IMAGE_REPO_TAG}-v_${BRANCH}_${BUILD_NUMBER} .
                    """
					}
				}

            }
        }



        stage('Push Docker Image') {
            steps {
                    sh("aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 264672321272.dkr.ecr.us-east-1.amazonaws.com/")
		   script { 
					if ("$GIT_BRANCH" == "main") {
						sh("docker push 264672321272.dkr.ecr.us-east-1.amazonaws.com//${DOCKER_IMAGE_REPO}:${DOCKER_IMAGE_REPO_TAG}-v_${GIT_BRANCH}_${BUILD_NUMBER}")	
					}
					else {
					    sh("docker push 264672321272.dkr.ecr.us-east-1.amazonaws.com/${DOCKER_IMAGE_REPO}:${DOCKER_IMAGE_REPO_TAG}-v_${BRANCH}_${BUILD_NUMBER}")
					}
			}

            }
        }
		stage ('Dev Deployment') {
			steps {
				script {
					sh """
					echo ${BUILD_NUMBER}
					cat ${APP_EKS}/${EKS_DEPLOY_YML}
					kubectl apply -f ${APP_EKS} --namespace csi-dev --context ${EKS_Cluster}
					"""
				}
			}
		}
		stage("Clean Workspace") {
			steps {
				cleanWs()
			}
		}
	}
}

