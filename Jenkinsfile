pipeline {
    agent {
        label 'maven-docker-executor'
    }

    options {
        timeout(time: 2, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '30'))
        disableConcurrentBuilds()
    }

    parameters {
        choice(
            name: 'DEPLOY_ENV', 
            choices: ['staging', 'production'], 
            description: 'Target Environment for deployment'
        )
    }

    environment {
        APP_NAME         = 'payment-gateway-service'
        REGISTRY_USER    = 'my-dockerhub-username'
        IMAGE_TAG        = "v_${env.BUILD_NUMBER}_${env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : 'local'}"
        
        SONAR_TOKEN      = credentials('sonarqube-analysis-token')
        DOCKER_HUB_CRED  = credentials('dockerhub-login-credentials')
    }

    stages {
        stage('Initialize & Clean') {
            steps {
                echo "Starting build sequence for ${env.APP_NAME}..."
                sh 'mvn clean'
            }
        }

        stage('Compile & Unit Test') {
            steps {
                sh 'mvn package -DskipTests=true'
            }
        }

        stage('SonarQube Static Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh "mvn sonar:sonar -Dsonar.projectKey=${env.APP_NAME} -Dsonar.login=${env.SONAR_TOKEN}"
                }
            }
        }

        stage('Build Container Image') {
            steps {
                sh "docker build --no-cache -t ${env.REGISTRY_USER}/${env.APP_NAME}:${env.IMAGE_TAG} ."
            }
        }

        stage('Push to Container Registry') {
            steps {
                withEnv(["DOCKER_USER=${DOCKER_HUB_CRED_USR}", "DOCKER_PASS=${DOCKER_HUB_CRED_PSW}"]) {
                    sh "echo \$DOCKER_PASS | docker login --username \$DOCKER_USER --password-stdin"
                    sh "docker push ${env.REGISTRY_USER}/${env.APP_NAME}:${env.IMAGE_TAG}"
                }
            }
        }

        stage('Staging Deployment') {
            when {
                expression { params.DEPLOY_ENV == 'staging' }
            }
            steps {
                sh "sed -i 's|IMAGE_PLACEHOLDER|${env.REGISTRY_USER}/${env.APP_NAME}:${env.IMAGE_TAG}|g' k8s/deployment.yaml"
                sh "kubectl apply -f k8s/deployment.yaml --namespace=staging"
            }
        }

        stage('Production Manual Approval') {
            when {
                expression { params.DEPLOY_ENV == 'production' }
            }
            steps {
                input message: "Approve production deployment for ${env.REGISTRY_USER}/${env.APP_NAME}:${env.IMAGE_TAG}?", ok: "Release to Prod"
            }
        }

        stage('Production Deployment') {
            when {
                allOf {
                    branch 'main'
                    expression { params.DEPLOY_ENV == 'production' }
                }
            }
            steps {
                sh "sed -i 's|IMAGE_PLACEHOLDER|${env.REGISTRY_USER}/${env.APP_NAME}:${env.IMAGE_TAG}|g' k8s/deployment.yaml"
                sh "kubectl apply -f k8s/deployment.yaml --namespace=production"
            }
        }
    }

    post {
        always {
            cleanWs()
            sh "docker rmi ${env.REGISTRY_USER}/${env.APP_NAME}:${env.IMAGE_TAG} || true"
            sh "docker logout"
        }
        success {
            echo "Deployment succeeded."
        }
        failure {
            echo "Pipeline failed."
        }
    }
}
