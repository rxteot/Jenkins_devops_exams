pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = 'dockerhub-creds'
        GITHUB_CREDENTIALS    = 'github-creds'
        KUBECONFIG_CRED       = 'kubeconfig'
        SLACK_CREDENTIAL      = 'slack-token'
        SLACK_CHANNEL         = '#deployment'

        MOVIE_IMAGE = "rxteot/movie-service"
        CAST_IMAGE  = "rxteot/cast-service"
    }

    stages {

        stage('Build Started') {
            steps {
                slackSend(
                    channel: SLACK_CHANNEL,
                    tokenCredentialId: SLACK_CREDENTIAL,
                    message: "🚀 Build started for *${env.JOB_NAME}* on branch *${env.BRANCH_NAME}* (#${env.BUILD_NUMBER})"
                )
            }
        }

        stage('Checkout') {
            steps {
                slackSend(channel: SLACK_CHANNEL, tokenCredentialId: SLACK_CREDENTIAL,
                          message: "📥 Starting *Checkout* stage for ${env.JOB_NAME} #${env.BUILD_NUMBER}")

                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${env.BRANCH_NAME}"]],
                    userRemoteConfigs: [[
                        url: 'https://github.com/rxteot/Jenkins_devops_exams.git',
                        credentialsId: "${GITHUB_CREDENTIALS}"
                    ]]
                ])
            }
        }

        stage('Build Images') {
            parallel {

                stage('Build Movie Service') {
                    steps {
                        slackSend(channel: SLACK_CHANNEL, tokenCredentialId: SLACK_CREDENTIAL,
                                  message: "🔧 Building *Movie Service* image")

                        sh """
                          docker build -t ${MOVIE_IMAGE}:${env.BRANCH_NAME}-${env.BUILD_NUMBER} ./movie-service
                        """
                    }
                }

                stage('Build Cast Service') {
                    steps {
                        slackSend(channel: SLACK_CHANNEL, tokenCredentialId: SLACK_CREDENTIAL,
                                  message: "🔧 Building *Cast Service* image")

                        sh """
                          docker build -t ${CAST_IMAGE}:${env.BRANCH_NAME}-${env.BUILD_NUMBER} ./cast-service
                        """
                    }
                }
            }
        }

        stage('Push Images') {
            steps {
                slackSend(channel: SLACK_CHANNEL, tokenCredentialId: SLACK_CREDENTIAL,
                          message: "📤 Pushing Docker images to DockerHub")

                withCredentials([usernamePassword(
                    credentialsId: "${DOCKERHUB_CREDENTIALS}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                      echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                      docker push ${MOVIE_IMAGE}:${env.BRANCH_NAME}-${env.BUILD_NUMBER}
                      docker push ${CAST_IMAGE}:${env.BRANCH_NAME}-${env.BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Deploy') {
            steps {
                slackSend(channel: SLACK_CHANNEL, tokenCredentialId: SLACK_CREDENTIAL,
                          message: "🚢 Starting *Helm Deployment* for branch ${env.BRANCH_NAME}")

                withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG')]) {
                    script {
                        def helmFlags = "--atomic --timeout 5m0s"

                        if (env.BRANCH_NAME == "dev" || env.BRANCH_NAME == "main" || env.BRANCH_NAME.startsWith("feature/")) {
                            sh """
                              export KUBECONFIG=${KUBECONFIG}
                              helm upgrade --install movie-platform-dev ./movie-platform \
                                -n dev \
                                -f movie-platform/values-dev.yaml \
                                --set movie_service.image.tag=${env.BRANCH_NAME}-${env.BUILD_NUMBER} \
                                --set cast_service.image.tag=${env.BRANCH_NAME}-${env.BUILD_NUMBER} \
                                ${helmFlags}
                            """
                        }

                        if (env.BRANCH_NAME == "qa") {
                            sh """
                              export KUBECONFIG=${KUBECONFIG}
                              helm upgrade --install movie-platform-qa ./movie-platform \
                                -n qa \
                                -f movie-platform/values-qa.yaml \
                                --set movie_service.image.tag=${env.BRANCH_NAME}-${env.BUILD_NUMBER} \
                                --set cast_service.image.tag=${env.BRANCH_NAME}-${env.BUILD_NUMBER} \
                                ${helmFlags}
                            """
                        }

                        if (env.BRANCH_NAME == "staging") {
                            sh """
                              export KUBECONFIG=${KUBECONFIG}
                              helm upgrade --install movie-platform-staging ./movie-platform \
                                -n staging \
                                -f movie-platform/values-staging.yaml \
                                --set movie_service.image.tag=${env.BRANCH_NAME}-${env.BUILD_NUMBER} \
                                --set cast_service.image.tag=${env.BRANCH_NAME}-${env.BUILD_NUMBER} \
                                ${helmFlags}
                            """
                        }

                        if (env.BRANCH_NAME == "master") {
                            timeout(time: 20, unit: 'MINUTES') {
                                input message: "Deploy to PRODUCTION?"
                            }
                            sh """
                              export KUBECONFIG=${KUBECONFIG}
                              helm upgrade --install movie-platform-prod ./movie-platform \
                                -n prod \
                                -f movie-platform/values-prod.yaml \
                                --set movie_service.image.tag=${env.BRANCH_NAME}-${env.BUILD_NUMBER} \
                                --set cast_service.image.tag=${env.BRANCH_NAME}-${env.BUILD_NUMBER} \
                                ${helmFlags}
                            """
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            slackSend(
                channel: SLACK_CHANNEL,
                tokenCredentialId: SLACK_CREDENTIAL,
                message: "✅ SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER} deployed from branch ${env.BRANCH_NAME}"
            )
        }
        failure {
            slackSend(
                channel: SLACK_CHANNEL,
                tokenCredentialId: SLACK_CREDENTIAL,
                message: "❌ FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER} on branch ${env.BRANCH_NAME}"
            )
        }
        always {
            echo "Pipeline completed with status: ${currentBuild.currentResult}"
        }
    }
}
