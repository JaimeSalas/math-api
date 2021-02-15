pipeline {
    agent any 
    parameters {
        booleanParam(name: 'CANARY_DEPLOYMENT', defaultValue: false, description: 'Deploy Canary?')
    }
    environment {
        imageName = 'jaimesalas/math-api'
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
                    def dockerImage = docker.build(imageName + ':latest')
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
            branch 'production'
          }
          steps {
            echo 'tag current version with v1 and push to registry'
            echo 'create docker immage with v2 from current code solution'
            script {
              if (params.CANARY_DEPLOYMENT) {
                withDockerRegistry([credentialsId: 'dockerhub-credentials', url: '']) {
                    sh '''
                      echo pulling latest and updating
                      docker pull ${imageName}:latest
                      docker tag ${imageName}:latest ${imageName}:v2
                      docker push ${imageName}:v2
                    '''
                }

                echo 'removing local images'
                sh 'docker rmi ${imageName}:latest'
                sh 'docker rmi ${imageName}:v2'
                
                sh 'echo connect to kubernetes and apply canary deployement...'
              } else {
                withDockerRegistry([credentialsId: 'dockerhub-credentials', url: '']) {
                    sh '''
                      echo pulling latest and updating
                      docker pull ${imageName}:latest
                      docker tag ${imageName}:latest ${imageName}:v1
                      docker push ${imageName}:v1
                    '''
                }

                // echo 'removing local images'
                // sh 'docker rmi ${imageName}:latest'
                // sh 'docker rmi ${imageName}:v1'
                cleanLocalImages(imageName, 'v1')
              }
            }
          }
        }
    }
}

void cleanLocalImages(String imageName, String version) {
  sh '''
    echo removing local images
    docker rmi ${imageName}:latest
    docker rmi ${imageName}:${version}
  '''
}