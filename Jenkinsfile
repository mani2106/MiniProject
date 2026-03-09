pipeline {
    agent any

    environment {
        IMAGE = "madhumitha3199/flask-app"
        TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Clone') {
            steps {
                git branch: 'main', 
                git 'https://github.com/RPMadhumitha/testjenkins.git',
                credentialsId: 'github-creds'
            }
        }

        stage('Build') {
            steps {
                script {
                    docker.build("${IMAGE}:${TAG}")
                }
            }
        }

        stage('Push') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                        docker.image("${IMAGE}:${TAG}").push()
                        docker.image("${IMAGE}:${TAG}").push('latest')
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sh "kubectl set image deployment/flask-app flask=${IMAGE}:${TAG}"
                sh "kubectl rollout status deployment/flask-app"
            }
        }
    }

    post {
        failure {
            sh "kubectl rollout undo deployment/flask-app"
        }
    }
}
