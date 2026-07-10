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
            choices: [
                'staging',
                'production'
            ],
            description: 'Choose deployment environment'
        )

    }


    environment {

        APP_NAME = "payment-gateway-service"

        DOCKER_USER = "ravibhadarge"

        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"

        IMAGE_TAG = "v${BUILD_NUMBER}-${GIT_COMMIT.take(7)}"

    }



    stages {


        stage('Checkout Code') {

            steps {

                checkout scm

                sh '''
                git rev-parse HEAD
                '''
            }

        }



        stage('Build Application') {

            steps {

                sh '''
                set -e

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


                sh '''

                docker build \
                -t ${IMAGE_NAME}:${IMAGE_TAG} .

                docker images ${IMAGE_NAME}

                '''

            }

        }




        stage('Docker Login') {

            steps {


                withCredentials([


                    usernamePassword(

                        credentialsId: 'dockerhub-login-credentials',

                        usernameVariable: 'USERNAME',

                        passwordVariable: 'PASSWORD'

                    )


                ]) {


                    sh '''

                    echo "$PASSWORD" | docker login \
                    -u "$USERNAME" \
                    --password-stdin

                    '''

                }

            }

        }





        stage('Docker Push') {

            steps {

                sh '''

                docker push ${IMAGE_NAME}:${IMAGE_TAG}

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

                sed "s|IMAGE_PLACEHOLDER|${IMAGE_NAME}:${IMAGE_TAG}|g" \
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

                    message: "Deploy ${IMAGE_TAG} to Production?",

                    ok: "Deploy"

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

                sed "s|IMAGE_PLACEHOLDER|${IMAGE_NAME}:${IMAGE_TAG}|g" \
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


            docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true


            '''


            cleanWs()

        }



        success {


            echo "SUCCESS: ${IMAGE_NAME}:${IMAGE_TAG}"

        }



        failure {


            echo "FAILED: Check previous stage logs"

        }


    }

}
