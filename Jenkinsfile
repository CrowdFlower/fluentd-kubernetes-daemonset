pipeline {
    agent { label 'docker' }

    environment {
        ECR_URL = "411719562396.dkr.ecr.us-east-1.amazonaws.com"
        APP_TAG = ""
        ALT_TAG = "build${env.BUILD_NUMBER}"
        GIT_COMMIT = ""
    }

    options {
        timestamps() // timestamp console output in Jenkins
        buildDiscarder(logRotator(artifactDaysToKeepStr: '1',
                                  artifactNumToKeepStr: '1',
                                  daysToKeepStr: '1',
                                  numToKeepStr: '1'))
    }

    stages {
        stage("Clean up dangling images") {
            steps {
                sh """
                    set +e
                    docker rm \$(docker ps -a -f status=exited -q) 2>/dev/null
                    docker rmi \$(docker images -f "dangling=true" -q) 2>/dev/null
                    set -e
                """
            }
        }
        stage('Build fluentd debian-nfs image') {
            steps {
                script {
                    GIT_COMMIT = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
                    STR_DATE = sh(returnStdout: true, script: "date +\"%y%m%d%H%M\"").trim()
                    APP_TAG = "${STR_DATE}-${GIT_COMMIT}"
                }

                sh """
                    docker build -t $ECR_URL/fluentd -f docker-image/v0.12/debian-nfs/Dockerfile docker-image/v0.12/debian-nfs
                """
            }
        }
        stage('Push Image to ECR') {
            steps {
                  echo "Pushing fluentd"
                  sh """
                      \$(aws ecr get-login --no-include-email --region us-east-1)
                      docker tag ${ECR_URL}/fluentd ${ECR_URL}/fluentd:${APP_TAG}
                      docker push ${ECR_URL}/fluentd:${APP_TAG}
                      docker tag ${ECR_URL}/fluentd ${ECR_URL}/fluentd:${ALT_TAG}
                      docker push ${ECR_URL}/fluentd:${ALT_TAG}
                  """
                  /* Tag latest build for qe0*
                  script {
                      qe_nums = ['00', '01', '02', '03', '04', '05']
                      qe_nums.each { num ->
                        sh """
                            docker tag ${ECR_URL}/fluentd ${ECR_URL}/fluentd:qe${num}
                            docker push ${ECR_URL}/fluentd:qe${num}
                        """
                      }
                  }
                  */
            }
        }
    }

    post {
        always {
            echo 'test cf jenkins pipeline'
        }

        failure {
            emailext (
                to: 'engineers@crowdflower.com',
                subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """
                <p>FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
                <p>Check console output at <a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a></p>
                """,
                recipientProviders: [[$class: 'DevelopersRecipientProvider']]
            )
        }
    }
}
