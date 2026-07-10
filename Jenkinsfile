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
            description: 'Select deployment environment'
        )

    }


    environment {

        APP_NAME = 'payment-gateway-service'

        DOCKER_REPO = 'ravibhadarge/payment-gateway-service'

        IMAGE_TAG = "v-${BUILD_NUMBER}"

        IMAGE = "${DOCKER_REPO}:${IMAGE_TAG}"

    }


    stages {


        stage('Checkout') {

            steps {

                checkout scm

                sh '''
                echo "Commit:"
                git rev-parse HEAD
                '''

            }

        }



        stage('Build') {

            steps {

                sh '''
                set -e

                mvn clean package -DskipTests=false

                '''

            }

        }



        stage('SonarQube Scan') {

            steps {


                withSonarQubeEnv('SonarQube-Server') {


                    withCredentials([

                        string(
                            credentialsId: 'sonarqube-analysis-token',
                            variable: 'SONAR_TOKEN'
                        )

                    ]) {


                        sh '''

                        set -e

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

                sh '''

                set -e

                docker build \
                -t ${IMAGE} .

                docker images ${DOCKER_REPO}

                '''

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

                    set -e


                    echo "$DOCKER_PASSWORD" | docker login \
                    -u "$DOCKER_USERNAME" \
                    --password-stdin


                    '''

                }

            }

        }




        stage('Docker Push') {

            steps {

                sh '''

                set -e

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

                set -e


                sed "s|IMAGE_PLACEHOLDER|${IMAGE}|g" \
                k8s/deployment.yaml > deployment.yaml


                kubectl apply \
                -f deployment.yaml \
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

                set -e


                sed "s|IMAGE_PLACEHOLDER|${IMAGE}|g" \
                k8s/deployment.yaml > deployment.yaml



                kubectl apply \
                -f deployment.yaml \
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

            echo "FAILED - check the stage above"

        }


    }

}
