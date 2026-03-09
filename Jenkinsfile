pipeline {
    agent any

    environment {
        IMAGE = "madhumitha3199/flask-app"
        TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Verify Setup') {
            steps {
                sh '''
                    echo "=== Verifying Docker Access ==="
                    docker --version
                    docker ps
                    echo "=== Verifying kubectl Access ==="
                    kubectl version --client
                    kubectl get nodes
                '''
            }
        }

        stage('Clone') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/RPMadhumitha/MiniProject.git',
                    credentialsId: 'github-creds'
            }
        }

        stage('Build') {
            steps {
                script {
                    docker.build("${IMAGE}:${TAG}", ".")
                }
            }
        }

        stage('Test Locally') {
            steps {
                sh '''
                    echo "Testing container before pushing..."
                    docker run --rm ${IMAGE}:${TAG} python -c "import flask; print('Flask installed OK')"
                '''
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
                sh """
                    echo "Deploying ${IMAGE}:${TAG}..."
                    kubectl set image deployment/flask-app flask=${IMAGE}:${TAG}
                    kubectl rollout status deployment/flask-app --timeout=2m
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "=== Checking Pods ==="
                    kubectl get pods -l app=flask
                    echo "=== Checking Service ==="
                    kubectl get svc flask-service
                    echo "=== Testing Pod Health ==="
                    kubectl run test-pod --rm -i --restart=Never --image=curlimages/curl --command -- curl -s http://flask-service/health || true
                '''
            }
        }
    }

    post {
        failure {
            sh '''
                echo "Deployment failed. Rolling back..."
                kubectl rollout undo deployment/flask-app
                echo "Rollback completed"
            '''
        }
        success {
            echo "Pipeline completed successfully! Access app with: minikube service flask-service"
        }
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
