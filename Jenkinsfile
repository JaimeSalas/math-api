pipeline {
    agent any 
    parameters {
        booleanParam(name: 'CanaryDeployment', defaultValue: false, description: 'Deploy Canary?')
    }
    environment {
        imageName = 'jaimesalas/math-api:latest'
        ec2Instance = 'ec2-15-236-142-40.eu-west-3.compute.amazonaws.com'
        appPort = 80
    }
    stages {
        stage('Install dependencies') {
            agent {
                docker {
                    image 'node:14-alpine'
                    reuseNode true
                }
            }
            steps {
                sh 'npm ci'
            }
        }
        stage('Tests') {
            agent {
                docker {
                    image 'node:14-alpine'
                    reuseNode true
                }
            }
            steps {
                sh 'npm test'
            }
        }
        stage('Staging Tests') {
            when {
                branch 'staging'
            }
            agent {
                docker {
                    image 'node:14-alpine'
                    reuseNode true
                }
            }
            environment {
                BASE_API_URL = "http://$ec2Instance:$appPort"
            }
            steps {
                sh 'npm run test:e2e'
            }
        }
        stage('Build image & push it to DockerHub') {
            when {
                branch 'develop'
            }
            steps {
                script {
                    def dockerImage = docker.build(imageName)
                    withDockerRegistry([credentialsId: 'dockerhub-credentials', url: '']) {
                        dockerImage.push()
                        sh 'docker rmi $imageName'
                    }
                }
            }
        }
        stage('Deploy to server') {
            when {
                branch 'develop'
            }
            environment {
                containerName = 'math-api'
            }
            steps {
                withCredentials(
                bindings: [sshUserPrivateKey(
                    credentialsId: 'ec2-ssh-credentials',
                    keyFileVariable: 'identityFile',
                    passphraseVariable: 'passphrase',
                    usernameVariable: 'user'
                )
                ]) {
                script {
                    sh '''
                        ssh -o StrictHostKeyChecking=no -i $identityFile $user@$ec2Instance \
                        APP_PORT=$appPort CONTAINER_NAME=$containerName IMAGE_NAME=$imageName bash < ./scripts/deploy.sh
                    '''
                    }
                }
            }
        }
        stage('Canary Deploy') {
          when {
            // branch 'production'
            expression {
              return GIT_BRANCH == 'production'
            }
          }
          steps {
            echo "${GIT_BRANCH}"
            echo 'tag current version with v1 and push to registry'
            echo 'create docker immage with v2 from current code solution'

          }
        }
    }
}