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

        DOCKER_REGISTRY = 'docker.io'

        REGISTRY_USER = 'ravibhadarge'

        IMAGE_NAME = "${REGISTRY_USER}/${APP_NAME}"

        IMAGE_TAG = "v${BUILD_NUMBER}-${GIT_COMMIT.take(7)}"

        SONAR_SERVER = 'SonarQube-Server'
    }


    stages {


        stage('Checkout') {

            steps {

                checkout scm

                sh '''
                echo "Git Commit:"
                git rev-parse HEAD
                '''
            }
        }



        stage('Initialize') {

            steps {

                sh '''
                echo "Application: ${APP_NAME}"

                java -version

                mvn -version

                docker --version

                kubectl version --client
                '''
            }
        }



        stage('Build Application') {

            steps {

                sh '''
                mvn clean package
                '''
            }
        }



        stage('SonarQube Scan') {

            steps {

                withSonarQubeEnv("${SONAR_SERVER}") {

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




        stage('Docker Hub Login') {

            steps {


                withCredentials([

                    usernamePassword(

                        credentialsId: 'dockerhub-login-credentials',

                        usernameVariable: 'DOCKER_USERNAME',

                        passwordVariable: 'DOCKER_TOKEN'

                    )

                ]) {


                    sh '''

                    echo "$DOCKER_TOKEN" | docker login \
                    --username "$DOCKER_USERNAME" \
                    --password-stdin


                    '''

                }
            }
        }





        stage('Push Docker Image') {

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

                kubectl create namespace staging \
                --dry-run=client -o yaml | kubectl apply -f -


                sed "s|IMAGE_PLACEHOLDER|${IMAGE_NAME}:${IMAGE_TAG}|g" \
                k8s/deployment.yaml > k8s/deployment-${BUILD_NUMBER}.yaml


                kubectl apply \
                -f k8s/deployment-${BUILD_NUMBER}.yaml \
                -n staging


                kubectl rollout status deployment/${APP_NAME} \
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

                    message: "Deploy ${IMAGE_TAG} to production?",

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


                kubectl create namespace production \
                --dry-run=client -o yaml | kubectl apply -f -



                sed "s|IMAGE_PLACEHOLDER|${IMAGE_NAME}:${IMAGE_TAG}|g" \
                k8s/deployment.yaml > k8s/deployment-${BUILD_NUMBER}.yaml



                kubectl apply \
                -f k8s/deployment-${BUILD_NUMBER}.yaml \
                -n production



                kubectl rollout status deployment/${APP_NAME} \
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


            echo "Deployment completed successfully: ${IMAGE_NAME}:${IMAGE_TAG}"


        }



        failure {


            echo "Pipeline failed"


        }


    }

}
