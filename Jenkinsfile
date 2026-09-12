// Jenkins Pipeline for Analytics Pvt Ltd's Capstone II lifecycle.
// Runs on every push to master (via the GitHub webhook) and is also scheduled for the 25th
// of each month, matching the org's release policy that master only ships on that date.
pipeline {
    agent any

    environment {
        IMAGE_NAME = "wuisman/capstone2-webapp"
    }

    triggers {
        githubPush()
        cron('H 0 25 * *')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker image') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} -t ${IMAGE_NAME}:latest ."
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                        docker push ${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'k3s-kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        export KUBECONFIG=$KUBECONFIG_FILE
                        kubectl apply -f k8s/webapp-deployment.yaml
                        kubectl apply -f k8s/webapp-service.yaml
                        kubectl set image deployment/webapp webapp=${IMAGE_NAME}:${BUILD_NUMBER}
                        kubectl rollout status deployment/webapp --timeout=120s
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Deployed ${IMAGE_NAME}:${BUILD_NUMBER} - 2 replicas behind NodePort 30008"
        }
        failure {
            echo "Pipeline failed - check the stage logs above"
        }
    }
}
