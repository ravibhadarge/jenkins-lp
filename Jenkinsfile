```groovy
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

        SONAR_TOKEN     = credentials('sonarqube-analysis-token')
        DOCKER_HUB_CRED = credentials('dockerhub-login-credentials')
    }


    stages {

        stage('Initialize & Clean') {
            steps {
                echo "Starting build sequence for ${env.APP_NAME}"

                sh '''
                    java -version
                    mvn -version
                    docker --version
                    mvn clean
                '''
            }
        }


        stage('Compile & Unit Test') {
            steps {
                echo "Building application"

                sh '''
                    mvn package
                '''
            }
        }


        stage('SonarQube Static Analysis') {
            steps {

                echo "Running SonarQube analysis"

                withSonarQubeEnv('SonarQube-Server') {

                    sh """
                    mvn sonar:sonar \
                    -Dsonar.projectKey=${APP_NAME} \
                    -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }


        stage('Build Container Image') {
            steps {

                echo "Building Docker image"

                sh """
                docker build \
                --no-cache \
                -t ${REGISTRY_USER}/${APP_NAME}:${IMAGE_TAG} .
                """
            }
        }


        stage('Push to Docker Hub') {
            steps {

                echo "Pushing image to Docker Hub"

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-login-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {

                    sh '''
                    echo "$DOCKER_PASS" | docker login \
                    --username "$DOCKER_USER" \
                    --password-stdin
                    '''

                    sh """
                    docker push ${REGISTRY_USER}/${APP_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }


        stage('Deploy to Staging') {

            when {
                expression {
                    params.DEPLOY_ENV == 'staging'
                }
            }

            steps {

                echo "Deploying to staging"

                sh """
                sed -i \
                's|IMAGE_PLACEHOLDER|${REGISTRY_USER}/${APP_NAME}:${IMAGE_TAG}|g' \
                k8s/deployment.yaml
                """

                sh """
                kubectl apply \
                -f k8s/deployment.yaml \
                --namespace=staging
                """

                sh """
                kubectl rollout status \
                deployment/${APP_NAME} \
                --namespace=staging
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
                    message: "Approve production deployment for ${REGISTRY_USER}/${APP_NAME}:${IMAGE_TAG}?",
                    ok: "Release to Production"
                )
            }
        }


        stage('Deploy to Production') {

            when {
                allOf {
                    branch 'main'
                    expression {
                        params.DEPLOY_ENV == 'production'
                    }
                }
            }

            steps {

                echo "Deploying to production"

                sh """
                sed -i \
                's|IMAGE_PLACEHOLDER|${REGISTRY_USER}/${APP_NAME}:${IMAGE_TAG}|g' \
                k8s/deployment.yaml
                """

                sh """
                kubectl apply \
                -f k8s/deployment.yaml \
                --namespace=production
                """

                sh """
                kubectl rollout status \
                deployment/${APP_NAME} \
                --namespace=production
                """
            }
        }
    }


    post {

        always {

            echo "Cleaning workspace"

            cleanWs()

            sh """
            docker rmi \
            ${REGISTRY_USER}/${APP_NAME}:${IMAGE_TAG} || true
            """

            sh """
            docker logout || true
            """
        }


        success {

            echo "Deployment succeeded successfully."
        }


        failure {

            echo "Pipeline failed. Check console logs."
        }
    }
}
```
