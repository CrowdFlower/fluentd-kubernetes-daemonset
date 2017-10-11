pipeline {
  agent { label 'docker' }

  options {
    buildDiscarder(logRotator(artifactDaysToKeepStr: '1', artifactNumToKeepStr: '1', daysToKeepStr: '1', numToKeepStr: '1'))
  }

  environment {
    IMAGE_NAME = 'fluentd'
    BRANCH_BUILD_TAG = "${BRANCH_NAME.replace('/','-')}-${BUILD_NUMBER}"
    GIT_SHA7 = "${GIT_COMMIT[0..6]}"
    TIMESTAMP = "${new Date(currentBuild.startTimeInMillis).format("yyMMddHHmm")}"
    DATE_SHA_TAG = "v0.12-debian-nfs-$TIMESTAMP-$GIT_SHA7"
    ECR_URL = '411719562396.dkr.ecr.us-east-1.amazonaws.com'
    IMAGE_REPO = "$ECR_URL/$IMAGE_NAME"
    CURRENT_BUILD_IMAGE_NAME = "$IMAGE_REPO:$BRANCH_BUILD_TAG"
    TIMESTAMPED_IMAGE_NAME = "$IMAGE_REPO:$DATE_SHA_TAG"
    ERROR_EMAIL = 'engineers@crowdflower.com'
  }

  stages {
    stage('Set error email') {
      when { expression { env.BRANCH_NAME != 'master' } }
      steps {
        script {
          ERROR_EMAIL = sh(returnStdout: true, script: "git --no-pager show -s --format='%ae' ${GIT_COMMIT}")
        }
        echo "Send topic branch build errors to commit author ${ERROR_EMAIL}"
      }
    }

    stage('Build and tag image') {
      parallel {
        stage('Clean up dangling images and containers') {
          steps {
            sh """
              set +e
              docker rm \$(docker ps -a -f status=exited -q) 2>/dev/null
              docker rm \$(docker ps -a -f status=dead -q) 2>/dev/null
              docker rmi \$(docker images -f "dangling=true" -q) 2>/dev/null
              set -e
            """
          }
        }
        stage('Build docker image') {
          steps {
            sh """
              docker build --no-cache \
                           -t ${CURRENT_BUILD_IMAGE_NAME} \
                           -t ${TIMESTAMPED_IMAGE_NAME} \
                           -f docker-image/v0.12/debian-nfs/Dockerfile \
                           docker-image/v0.12/debian-nfs
            """
          }
        }
      }
    }

    stage('Push Image to ECR') {
      steps {
        echo "Pushing ${IMAGE_NAME} tagged ${BRANCH_BUILD_TAG} and ${DATE_SHA_TAG}"
        sh """
          \$(aws ecr get-login --no-include-email --region us-east-1)
          docker push ${CURRENT_BUILD_IMAGE_NAME}
          docker push ${TIMESTAMPED_IMAGE_NAME}
        """
        /* Tag latest build for qe0*
        script {
            qe_nums = ['00', '01', '02', '03', '04', '05']
            qe_nums.each { num ->
              sh """
                  docker tag ${CURRENT_BUILD_IMAGE_NAME} ${IMAGE_REPO}:qe${num}
                  docker push ${IMAGE_REPO}:qe${num}
              """
            }
        }
        */
      }
    }
  } /* stages */

  post {
    always {
      echo 'Clean up any built/tagged images'
      sh """
        docker rmi -f ${TIMESTAMPED_IMAGE_NAME} >/dev/null 2>&1
        docker rmi -f ${CURRENT_BUILD_IMAGE_NAME} >/dev/null 2>&1
      """
    }

    failure {
      emailext (
        to: "${ERROR_EMAIL}",
        subject: "${currentBuild.currentResult}: ${currentBuild.fullDisplayName} - template html-with-health-and-console.jelly",
        body: '''${JELLY_SCRIPT,template="html-with-health-and-console"}''',
        recipientProviders: [[$class: 'DevelopersRecipientProvider']]
      )
    }
  }
}
