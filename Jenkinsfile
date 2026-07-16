pipeline {
    agent any

    environment {
        DOCKER_ID = "farhadwais"
        DOCKER_TAG = "v.${BUILD_NUMBER}.0"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build images') {
            steps {
                sh '''
                docker build -t $DOCKER_ID/cast-service:$DOCKER_TAG ./cast-service
                docker build -t $DOCKER_ID/movie-service:$DOCKER_TAG ./movie-service
                '''
            }
        }

        stage('Test images') {
            steps {
                sh '''
                docker rm -f cast-test || true
                docker run -d --name cast-test -p 8100:8000 $DOCKER_ID/cast-service:$DOCKER_TAG
                sleep 8
                curl -f http://localhost:8100/api/v1/casts/docs
                docker rm -f cast-test
                '''
                sh '''
                docker rm -f movie-test || true
                docker run -d --name movie-test -p 8101:8000 $DOCKER_ID/movie-service:$DOCKER_TAG
                sleep 8
                curl -f http://localhost:8101/api/v1/movies/docs
                docker rm -f movie-test
                '''
            }
        }

        stage('Push images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                    docker push $DOCKER_ID/cast-service:$DOCKER_TAG
                    docker push $DOCKER_ID/movie-service:$DOCKER_TAG
                    '''
                }
            }
        }

        stage('Deploy dev') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-credentials', variable: 'KUBECONFIG')]) {
                    sh '''
                    helm upgrade --install cast-service ./cast-chart --namespace dev --set image.tag=$DOCKER_TAG
                    helm upgrade --install movie-service ./movie-chart --namespace dev --set image.tag=$DOCKER_TAG
                    '''
                }
            }
        }

        stage('Deploy qa') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-credentials', variable: 'KUBECONFIG')]) {
                    sh '''
                    helm upgrade --install cast-service ./cast-chart --namespace qa --set image.tag=$DOCKER_TAG
                    helm upgrade --install movie-service ./movie-chart --namespace qa --set image.tag=$DOCKER_TAG
                    '''
                }
            }
        }

        stage('Deploy staging') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-credentials', variable: 'KUBECONFIG')]) {
                    sh '''
                    helm upgrade --install cast-service ./cast-chart --namespace staging --set image.tag=$DOCKER_TAG
                    helm upgrade --install movie-service ./movie-chart --namespace staging --set image.tag=$DOCKER_TAG
                    '''
                }
            }
        }

        stage('Deploy prod approval') {
            when {
                branch 'master'
            }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: "Deploy build $DOCKER_TAG to PRODUCTION?", ok: "Deploy"
                }
            }
        }

        stage('Deploy prod') {
            when {
                branch 'master'
            }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-credentials', variable: 'KUBECONFIG')]) {
                    sh '''
                    helm upgrade --install cast-service ./cast-chart --namespace prod --set image.tag=$DOCKER_TAG
                    helm upgrade --install movie-service ./movie-chart --namespace prod --set image.tag=$DOCKER_TAG
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker rm -f cast-test movie-test || true'
        }
    }
}
