pipeline {

    agent any

    options {
        timeout(time: 2, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '30'))
        disableConcurrentBuilds()
        timestamps()
    }

    parameters {
        choice(
            name: 'DEPLOY_ENV',
            choices: ['staging', 'production'],
            description: 'Target Environment for deployment'
        )
    }

    environment {
        APP_NAME        = 'payment-gateway-service'
        REGISTRY_USER   = 'ravibhadarge'
        IMAGE_TAG       = "v_${env.BUILD_NUMBER}_${env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : 'local'}"
    }

    stages {

        stage('Initialize') {
            steps {
                echo "Starting ${APP_NAME}"

                sh '''
                    java -version
                    mvn -version
                    docker --version
                '''
            }
        }


        stage('Clean and Build') {
            steps {
                sh '''
                    mvn clean package -DskipTests=false
                '''
            }
        }


        stage('SonarQube Analysis') {
            steps {

                withSonarQubeEnv('SonarQube-Server') {

                    withCredentials([
                        string(
                            credentialsId: 'sonarqube-analysis-token',
                            variable: 'SONAR_TOKEN'
                        )
                    ]) {

                        sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=${APP_NAME} \
                        -Dsonar.login=${SONAR_TOKEN}
                        '''
                    }
                }
            }
        }


        stage('Docker Build') {
            steps {

                sh """
                docker build \
                -t ${REGISTRY_USER}/${APP_NAME}:${IMAGE_TAG} .
                """
            }
        }


        stage('Docker Login and Push') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-login-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {

                    sh '''
                    echo "$DOCKER_PASS" | docker login \
                    -u "$DOCKER_USER" \
                    --password-stdin
                    '''

                    sh """
                    docker push ${REGISTRY_USER}/${APP_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }


        stage('Deploy Staging') {

            when {
                expression {
                    params.DEPLOY_ENV == 'staging'
                }
            }

            steps {

                sh """
                sed -i \
                's|IMAGE_PLACEHOLDER|${REGISTRY_USER}/${APP_NAME}:${IMAGE_TAG}|g' \
                k8s/deployment.yaml
                """

                sh """
                kubectl apply \
                -f k8s/deployment.yaml \
                -n staging
                """
            }
        }


        stage('Production Approval') {

            when {
                expression {
                    params.DEPLOY_ENV == 'production'
                }
            }

            steps {

                input(
                    message: "Deploy ${IMAGE_TAG} to production?",
                    ok: "Approve"
                )
            }
        }


        stage('Deploy Production') {

            when {
                allOf {
                    branch 'main'
                    expression {
                        params.DEPLOY_ENV == 'production'
                    }
                }
            }

            steps {

                sh """
                sed -i \
                's|IMAGE_PLACEHOLDER|${REGISTRY_USER}/${APP_NAME}:${IMAGE_TAG}|g' \
                k8s/deployment.yaml
                """

                sh """
                kubectl apply \
                -f k8s/deployment.yaml \
                -n production
                """
            }
        }
    }


    post {

        always {

            sh """
            docker rmi \
            ${REGISTRY_USER}/${APP_NAME}:${IMAGE_TAG} || true
            """

            sh "docker logout || true"

            cleanWs()
        }


        success {
            echo "Pipeline completed successfully"
        }


        failure {
            echo "Pipeline failed"
        }
    }
}
