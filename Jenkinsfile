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
            description: 'Deployment Environment'
        )

    }



    environment {

        APP_NAME = 'payment-gateway-service'

        DOCKER_IMAGE = 'ravibhadarge/payment-gateway-service'

        PATH = "/usr/local/bin:${env.PATH}"

        KUBECONFIG = "/var/lib/jenkins/.kube/config"

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


                    env.IMAGE_TAG =
                    "v-${BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}"


                    env.IMAGE =
                    "${DOCKER_IMAGE}:${env.IMAGE_TAG}"


                    echo "Image: ${env.IMAGE}"

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
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )

                ]) {


                    sh '''

                    echo "$DOCKER_PASS" | docker login \
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

                docker images ${DOCKER_IMAGE}

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





        stage('Kind Kubernetes Check') {

            steps {

                sh '''

                export PATH=/usr/local/bin:$PATH


                echo "Kubernetes Context"

                kubectl config current-context


                echo "Kubernetes Nodes"

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

                export PATH=/usr/local/bin:$PATH


                kubectl create namespace staging \
                --dry-run=client -o yaml | kubectl apply -f -



                sed \
                "s|IMAGE_PLACEHOLDER|${IMAGE}|g" \
                k8s/deployment.yaml \
                > deployment.yaml



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

                export PATH=/usr/local/bin:$PATH


                kubectl create namespace production \
                --dry-run=client -o yaml | kubectl apply -f -



                sed \
                "s|IMAGE_PLACEHOLDER|${IMAGE}|g" \
                k8s/deployment.yaml \
                > deployment.yaml



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

            echo "FAILED: Check previous stage logs"

        }


    }

}
