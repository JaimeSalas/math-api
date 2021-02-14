pipeline {
    agent any 
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
        stage('Build') {
            agent {
                docker {
                    image 'node:14-alpine'
                    reuseNode true
                }
            }
            steps {
                sh 'npm run build'
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
    }
}