pipeline {

    agent {
        label 'maven-docker-executor'
    }


    options {

        timeout(time: 2, unit: 'HOURS')

        buildDiscarder(logRotator(numToKeepStr: '30'))

        disableConcurrentBuilds()

        timestamps()

    }



    parameters {

        choice(
            name: 'DEPLOY_ENV',
            choices: [
                'staging',
                'production'
            ],
            description: 'Deployment Environment'
        )

    }



    environment {

        APP_NAME = 'payment-gateway-service'

        DOCKER_REPOSITORY = 'ravibhadarge/payment-gateway-service'

    }



    stages {


        stage('Checkout') {

            steps {

                checkout scm


                script {

                    env.GIT_COMMIT_SHORT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()


                    env.IMAGE_TAG = "v-${BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}"

                    env.IMAGE =
                    "${DOCKER_REPOSITORY}:${env.IMAGE_TAG}"


                    echo "Building image: ${env.IMAGE}"

                }

            }

        }



        stage('Maven Build') {

            steps {

                sh '''

                mvn clean package

                '''

            }

        }



        stage('Unit Test') {

            steps {

                sh '''

                mvn test

                '''

            }


            post {

                always {

                    junit(
                        testResults: 'target/surefire-reports/*.xml',
                        allowEmptyResults: true
                    )

                }

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
                        -Dsonar.token=${SONAR_TOKEN}

                        '''

                    }

                }

            }

        }





        stage('Docker Login') {

            steps {


                withCredentials([

                    usernamePassword(
                        credentialsId: 'dockerhub-login-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )

                ]) {


                    sh '''

                    echo "$DOCKER_PASSWORD" | docker login \
                    -u "$DOCKER_USERNAME" \
                    --password-stdin

                    '''

                }

            }

        }





        stage('Docker Build') {

            steps {

                sh '''

                docker build \
                -t ${IMAGE} .

                '''

            }

        }





        stage('Docker Push') {

            steps {

                sh '''

                docker push ${IMAGE}

                '''

            }

        }





        stage('Deploy Staging') {


            when {

                expression {

                    params.DEPLOY_ENV == 'staging'

                }

            }



            steps {


                sh '''

                sed \
                "s|IMAGE_PLACEHOLDER|${IMAGE}|g" \
                k8s/deployment.yaml > deployment-staging.yaml



                kubectl apply \
                -f deployment-staging.yaml \
                -n staging



                kubectl rollout status \
                deployment/${APP_NAME} \
                -n staging

                '''

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

                    message: "Deploy ${IMAGE} to production?",

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


                sh '''

                sed \
                "s|IMAGE_PLACEHOLDER|${IMAGE}|g" \
                k8s/deployment.yaml > deployment-production.yaml



                kubectl apply \
                -f deployment-production.yaml \
                -n production



                kubectl rollout status \
                deployment/${APP_NAME} \
                -n production

                '''

            }

        }


    }




    post {


        always {


            sh '''

            docker logout || true


            docker rmi ${IMAGE} || true


            '''


            cleanWs()

        }



        success {

            echo "SUCCESS: ${IMAGE}"

        }



        failure {

            echo "FAILED: Review the stage logs above"

        }


    }

}
