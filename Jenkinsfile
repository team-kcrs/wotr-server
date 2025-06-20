pipeline {
    agent any

    environment {
        SPRING_PROFILE = "prod"

        // AWS ECR
        AWS_REGION = "ap-northeast-2"
        ECR_REPO = "688567260818.dkr.ecr.ap-northeast-2.amazonaws.com/wotr-ecr"
        IMAGE_NAME="jenkins-with-awscli"
    }

    options {
        skipDefaultCheckout(true)
    }

    stages {
        stage('Clean Workspace') {
            steps {
                deleteDir() // workspace clean
            }
        }
        stage('Checkout') {
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/team-kcrs/wotr-server.git',
                        credentialsId: 'wotr-git-credentials'
                    ]]
                ])
            }
        }

        stage('Prepare .env') {
            steps {
                withCredentials([file(credentialsId: 'wotr-server-env-file', variable: 'ENV_PATH')]) {
                    sh '''
                      if [ -f "$WORKSPACE/.env" ]; then
                        echo "[DEBUG] Existing .env:"
                        ls -l "$WORKSPACE/.env"
                      fi

                      rm -f "$WORKSPACE/.env"
                      cp "$ENV_PATH" "$WORKSPACE/.env"
                    '''
                }
            }
        }

        stage('Build') {
            steps {
                sh '''
                    chmod +x gradlew
                    export $(cat .env | xargs)
                    ./gradlew clean build -Dspring.profiles.active=$SPRING_PROFILE
                '''
            }
        }

        stage('Set Image Tag') {
            steps {
                script {
                    def timestamp = sh(script: "date +%Y%m%d%H%M%S", returnStdout: true).trim()
                    def shortGitCommit = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    env.IMAGE_TAG = "${timestamp}-${shortGitCommit}"
                    echo "Generated IMAGE_TAG: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t $IMAGE_NAME .
                    docker tag $IMAGE_NAME:latest $ECR_REPO:$IMAGE_TAG
                '''
            }
        }

        stage('Login & Push to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'wotr-aws-ecr-credentials'
                ]]) {
                    sh '''
                        aws ecr get-login-password --region $AWS_REGION \
                        | docker login --username AWS --password-stdin $ECR_REPO
                        docker push $ECR_REPO:$IMAGE_TAG
                    '''
                }
            }
        }

        stage('Fetch Env From S3') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'wotr-aws-s3-credentials'
                ]]) {
                    sh '''
                        aws s3 cp s3://wotr-server-s3/.env .env
                    '''
                }
            }
        }

        stage('Parse Env and Register Task Definition') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'wotr-aws-ecs-credentials'
                ]]) {
                    script {
                        def envMap = [:]
                        def envLines = readFile('.env').split("\n")
                        envLines.each {
                            if (it.trim() && it.contains('=')) {
                                def (key, value) = it.split('=', 2)
                                envMap[key.trim()] = value.trim()
                            }
                        }

                        def envJson = envMap.collect { key, val ->
                            """{ "name": "${key}", "value": "${val}" }"""
                        }.join(',')

                        def taskDefJson = """
                        {
                          "family": "wotr-fargate-task",
                          "networkMode": "awsvpc",
                          "executionRoleArn": "arn:aws:iam::688567260818:role/ecsTaskExecutionRole",
                          "requiresCompatibilities": ["FARGATE"],
                          "cpu": "512",
                          "memory": "1024",
                          "containerDefinitions": [
                            {
                              "name": "wotr-server-fargate",
                              "image": "${env.ECR_REPO}:${env.IMAGE_TAG}",
                              "portMappings": [
                                {
                                  "containerPort": 8080,
                                  "protocol": "tcp"
                                }
                              ],
                              "essential": true,
                              "environment": [
                                ${envJson}
                              ]
                            }
                          ]
                        }
                        """

                        writeFile file: 'task-definition.json', text: taskDefJson

                        sh '''
                            aws ecs register-task-definition \
                                --cli-input-json file://task-definition.json \
                                --region $AWS_REGION
                        '''
                    }
                }
            }
        }
    }
}
