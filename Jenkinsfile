pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "sanikanaiknaware/netflix-app"
        AWS_REGION = "ap-south-1"
        EKS_CLUSTER = "netflix-cluster"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    corepack enable || true
                    yarn install --frozen-lockfile
                '''
            }
        }

        stage('Build Application') {
            steps {
                sh 'yarn build'
            }
        }

        stage('Configure EKS') {
            steps {
                sh '''
                    aws eks update-kubeconfig \
                      --region $AWS_REGION \
                      --name $EKS_CLUSTER

                    kubectl get nodes
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'tmdb-api-key',
                        variable: 'TMDB_API_KEY'
                    )
                ]) {
                    sh '''
                        docker build \
                          --build-arg TMDB_V3_API_KEY="$TMDB_API_KEY" \
                          -t $DOCKER_IMAGE:$BUILD_NUMBER \
                          -t $DOCKER_IMAGE:latest .
                    '''
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKERHUB_USERNAME',
                        passwordVariable: 'DOCKERHUB_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$DOCKERHUB_PASSWORD" | docker login \
                          -u "$DOCKERHUB_USERNAME" \
                          --password-stdin

                        docker push $DOCKER_IMAGE:$BUILD_NUMBER
                        docker push $DOCKER_IMAGE:latest
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    kubectl set image deployment/netflix-app \
                      netflix-app=$DOCKER_IMAGE:$BUILD_NUMBER

                    kubectl rollout status deployment/netflix-app \
                      --timeout=5m
                '''
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }

        success {
            echo 'Netflix application deployed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check the Jenkins Console Output.'
        }
    }
}
