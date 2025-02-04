pipeline {

    agent any

    tools {
        go 'Go 1.18'
    }

    environment {
      RELEASE_TYPE = 'release'
      GIT_CREDENTIAL_ID = 'wf-jenkins-github'
      GITHUB_TOKEN = credentials('GITHUB_TOKEN')
      REPO_NAME = 'wavefront-kubernetes-adapter'
  }

    parameters {
        string(name: 'VERSION_NUMBER', defaultValue: '', description: 'The version number to release as')
        string(name: 'TARGET_COMMITISH', defaultValue: 'master', description: 'Specify a specific commit hash or branch to release the version to.')
        string(name: 'RELEASE_NOTES', defaultValue: '', description: 'The public release notes for the version. Use \\n to create newlines')
        booleanParam(name: 'IS_DRAFT', defaultValue: true, description: 'If the release should be marked as a draft (unpublished)')
        booleanParam(name: 'IS_PRERELEASE', defaultValue: false, description: 'If the release should be marked as a prerelease')
        booleanParam(name: 'createGithubRelease', defaultValue: true, description: 'Mark as false if you only want to build/push docker images. Note: a tag specified by the name VERSION must be created in the repo already for this option to be turned off')
    }

    stages {
        stage("Publish GA Harbor Image") {
          environment {
            HARBOR_CREDS = credentials("projects-registry-vmware-tanzu_observability-robot")
            DOCKER_REPO = 'projects.registry.vmware.com/tanzu_observability'
            DOCKER_IMAGE = 'kubernetes-hpa-adapter'
          }
          steps {
            sh 'echo $HARBOR_CREDS_PSW | docker login $DOCKER_REPO -u $HARBOR_CREDS_USR --password-stdin'
            sh './scripts/install_docker_buildx.sh'
            sh 'make publish'
          }
        }

        stage("Publish GA Docker Hub") {
          environment {
            DOCKERHUB_CREDS=credentials('Dockerhub_svcwfjenkins')
            DOCKER_REPO = 'wavefronthq'
            DOCKER_IMAGE = 'wavefront-hpa-adapter'
          }

          steps {
            sh 'echo $DOCKERHUB_CREDS_PSW | docker login -u $DOCKERHUB_CREDS_USR --password-stdin'
            sh './scripts/install_docker_buildx.sh'
            sh 'make publish'
          }
        }

        stage('Create Github Release') {
            steps {
                script {
                    if (params.createGithubRelease) {
                        TARGET_COMMITISH_TRIMMED = TARGET_COMMITISH.minus("origin/")
                        sh "curl -XPOST -H \"Authorization: token ${GITHUB_TOKEN}\" -H \"Accept: application/vnd.github.v3+json\" https://api.github.com/repos/wavefrontHQ/${REPO_NAME}/releases -d \'{\"tag_name\": \"${VERSION_NUMBER}\", \"target_commitish\": \"${TARGET_COMMITISH_TRIMMED}\", \"body\": \"${RELEASE_NOTES}\", \"draft\": ${IS_DRAFT}, \"prerelease\": ${IS_PRERELEASE}}\'"
                    }
                }
            }
        }
    }

    post {
      // Notify only on null->failure or success->failure or any->success
      failure {
        script {
          if(currentBuild.previousBuild == null) {
            slackSend (channel: '#tobs-k8po-team', color: '#FF0000', message: "RELEASE BUILD FAILED: <${env.BUILD_URL}|${env.JOB_NAME} [${env.BUILD_NUMBER}]>")
          }
        }
      }
      regression {
        slackSend (channel: '#tobs-k8po-team', color: '#FF0000', message: "RELEASE BUILD FAILED: <${env.BUILD_URL}|${env.JOB_NAME} [${env.BUILD_NUMBER}]>")
      }
      success {
        script {
          slackSend (channel: '#tobs-k8s-assist', color: '#008000', message: "Success!! `wavefront-kubernetes-adapter:${VERSION_NUMBER}` released!")
        }
      }
      always {
        cleanWs()
      }
    }
}
