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

        DOCKER_IMAGE = 'ravibhadarge/payment-gateway-service'

        KUBECONFIG = credentials('kind-kubeconfig')

    }



    stages {


        stage('Checkout') {

            steps {

                checkout scm


                script {

                    env.COMMIT_ID = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()


                    env.IMAGE_TAG =
                    "v-${BUILD_NUMBER}-${env.COMMIT_ID}"


                    env.IMAGE =
                    "${DOCKER_IMAGE}:${env.IMAGE_TAG}"


                    echo "IMAGE=${env.IMAGE}"

                }

            }

        }




        stage('Build') {

            steps {

                sh '''

                mvn clean package -DskipTests=false

                '''

            }

        }





        stage('Test') {

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
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )

                ]) {


                    sh '''

                    echo "$DOCKER_PASSWORD" | docker login \
                    -u "$DOCKER_USER" \
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





        stage('Kubernetes Connection Test') {

            steps {

                sh '''

                echo "Kubernetes Context"

                kubectl config current-context


                echo "Cluster Nodes"

                kubectl get nodes


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

                kubectl create namespace production \
                --dry-run=client -o yaml | kubectl apply -f -



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

            echo "Deployment Successful: ${IMAGE}"

        }



        failure {

            echo "Pipeline Failed"

        }


    }

}
