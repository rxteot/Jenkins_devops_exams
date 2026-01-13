pipeline {
    agent any

    environment {
        // Credentials
        DOCKERHUB_CREDENTIALS = 'dockerhub-creds'
        GITHUB_CREDENTIALS    = 'github-creds'
        KUBECONFIG_CRED       = 'kubeconfig'

        // Slack
        SLACK_CHANNEL = '#deployment'
        SLACK_CREDENTIAL = 'slack-token'   // Slack Bot/User OAuth Token stored in Jenkins

        // Images
        MOVIE_IMAGE = "rxteot/movie-service"
        CAST_IMAGE  = "rxteot/cast-service"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${BRANCH_NAME}"]],
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
                        sh '''
                          docker build -t ${MOVIE_IMAGE}:${BRANCH_NAME}-${BUILD_NUMBER} ./movie-service
                        '''
                    }
                }

                stage('Build Cast Service') {
                    steps {
                        sh '''
                          docker build -t ${CAST_IMAGE}:${BRANCH_NAME}-${BUILD_NUMBER} ./cast-service
                        '''
                    }
                }
            }
        }

        stage('Push Images') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKERHUB_CREDENTIALS}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                      echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                      docker push ${MOVIE_IMAGE}:${BRANCH_NAME}-${BUILD_NUMBER}
                      docker push ${CAST_IMAGE}:${BRANCH_NAME}-${BUILD_NUMBER}
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG')]) {
                    script {
                        def helmFlags = "--atomic --timeout 5m0s"

                        if (BRANCH_NAME == "dev" || BRANCH_NAME == "main" || BRANCH_NAME.startsWith("feature/")) {
                            sh """
                              export KUBECONFIG=${KUBECONFIG}
                              helm upgrade --install movie-platform-dev ./movie-platform \
                                -n dev \
                                -f movie-platform/values-dev.yaml \
                                --set movie_service.image.tag=${BRANCH_NAME}-${BUILD_NUMBER} \
                                --set cast_service.image.tag=${BRANCH_NAME}-${BUILD_NUMBER} \
                                ${helmFlags}
                            """
                        }

                        if (BRANCH_NAME == "qa") {
                            sh """
                              export KUBECONFIG=${KUBECONFIG}
                              helm upgrade --install movie-platform-qa ./movie-platform \
                                -n qa \
                                -f movie-platform/values-qa.yaml \
                                --set movie_service.image.tag=${BRANCH_NAME}-${BUILD_NUMBER} \
                                --set cast_service.image.tag=${BRANCH_NAME}-${BUILD_NUMBER} \
                                ${helmFlags}
                            """
                        }

                        if (BRANCH_NAME == "staging") {
                            sh """
                              export KUBECONFIG=${KUBECONFIG}
                              helm upgrade --install movie-platform-staging ./movie-platform \
                                -n staging \
                                -f movie-platform/values-staging.yaml \
                                --set movie_service.image.tag=${BRANCH_NAME}-${BUILD_NUMBER} \
                                --set cast_service.image.tag=${BRANCH_NAME}-${BUILD_NUMBER} \
                                ${helmFlags}
                            """
                        }

                        if (BRANCH_NAME == "master") {
                            timeout(time: 20, unit: 'MINUTES') {
                                input message: "Deploy to PRODUCTION?"
                            }
                            sh """
                              export KUBECONFIG=${KUBECONFIG}
                              helm upgrade --install movie-platform-prod ./movie-platform \
                                -n prod \
                                -f movie-platform/values-prod.yaml \
                                --set movie_service.image.tag=${BRANCH_NAME}-${BUILD_NUMBER} \
                                --set cast_service.image.tag=${BRANCH_NAME}-${BUILD_NUMBER} \
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
                channel: '#deployment',
                tokenCredentialId: 'slack-token',
                message: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER} deployed from branch ${env.BRANCH_NAME}"
            )
        }
        failure {
            slackSend(
                channel: '#deployment',
                tokenCredentialId: 'slack-token',
                message: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER} on branch ${env.BRANCH_NAME}"
            )
        }
        always {
             slackSend(
                channel: '#deployment',
                tokenCredentialId: SLACK_CREDENTIAL,
                message: "Pipeline completed with status: ${currentBuild.currentResult}"
            )
            echo "Pipeline completed with status: ${currentBuild.currentResult}"
        }
    }

}
