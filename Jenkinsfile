pipeline {
    agent any

    options {
        disableConcurrentBuilds()
    }

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
                docker network rm test-net || true
                docker network create test-net

                docker rm -f cast-test-db || true
                docker run -d --name cast-test-db --network test-net \
                  -e POSTGRES_USER=cast_db_username \
                  -e POSTGRES_PASSWORD=cast_db_password \
                  -e POSTGRES_DB=cast_db_dev \
                  postgres:12.1-alpine
                sleep 10

                docker rm -f cast-test || true
                docker run -d --name cast-test --network test-net \
                  -e DATABASE_URI=postgresql://cast_db_username:cast_db_password@cast-test-db/cast_db_dev \
                  $DOCKER_ID/cast-service:$DOCKER_TAG \
                  uvicorn app.main:app --host 0.0.0.0 --port 8000 --loop asyncio
                sleep 8
                echo "----- cast-test-db logs -----"
                docker logs cast-test-db
                echo "----- cast-test logs -----"
                docker logs cast-test
                echo "----- cast-test status -----"
                docker inspect cast-test --format '{{.State.Status}} exitcode={{.State.ExitCode}}'
                docker exec cast-test python -c "import urllib.request; assert urllib.request.urlopen('http://localhost:8000/api/v1/casts/docs').status == 200; print('cast-service OK')"

                docker rm -f cast-test cast-test-db
                '''
                sh '''
                docker rm -f movie-test-db || true
                docker run -d --name movie-test-db --network test-net \
                  -e POSTGRES_USER=movie_db_username \
                  -e POSTGRES_PASSWORD=movie_db_password \
                  -e POSTGRES_DB=movie_db_dev \
                  postgres:12.1-alpine
                sleep 10

                docker rm -f movie-test || true
                docker run -d --name movie-test --network test-net \
                  -e DATABASE_URI=postgresql://movie_db_username:movie_db_password@movie-test-db/movie_db_dev \
                  -e CAST_SERVICE_HOST_URL=http://cast-test/api/v1/casts/ \
                  $DOCKER_ID/movie-service:$DOCKER_TAG \
                  uvicorn app.main:app --host 0.0.0.0 --port 8000 --loop asyncio
                sleep 8
                echo "----- movie-test-db logs -----"
                docker logs movie-test-db
                echo "----- movie-test logs -----"
                docker logs movie-test
                echo "----- movie-test status -----"
                docker inspect movie-test --format '{{.State.Status}} exitcode={{.State.ExitCode}}'
                docker exec movie-test python -c "import urllib.request; assert urllib.request.urlopen('http://localhost:8000/api/v1/movies/docs').status == 200; print('movie-service OK')"

                docker rm -f movie-test movie-test-db
                docker network rm test-net
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
                    helm upgrade --install cast-service ./cast-chart --namespace dev --set image.tag=$DOCKER_TAG --set service.nodePort=30081
                    helm upgrade --install movie-service ./movie-chart --namespace dev --set image.tag=$DOCKER_TAG --set service.nodePort=30082
                    '''
                }
            }
        }

        stage('Deploy qa') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-credentials', variable: 'KUBECONFIG')]) {
                    sh '''
                    helm upgrade --install cast-service ./cast-chart --namespace qa --set image.tag=$DOCKER_TAG --set service.nodePort=30083
                    helm upgrade --install movie-service ./movie-chart --namespace qa --set image.tag=$DOCKER_TAG --set service.nodePort=30084
                    '''
                }
            }
        }

        stage('Deploy staging') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-credentials', variable: 'KUBECONFIG')]) {
                    sh '''
                    helm upgrade --install cast-service ./cast-chart --namespace staging --set image.tag=$DOCKER_TAG --set service.nodePort=30085
                    helm upgrade --install movie-service ./movie-chart --namespace staging --set image.tag=$DOCKER_TAG --set service.nodePort=30086
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
                    helm upgrade --install cast-service ./cast-chart --namespace prod --set image.tag=$DOCKER_TAG --set service.nodePort=30087
                    helm upgrade --install movie-service ./movie-chart --namespace prod --set image.tag=$DOCKER_TAG --set service.nodePort=30088
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker rm -f cast-test movie-test cast-test-db movie-test-db || true'
            sh 'docker network rm test-net || true'
        }
    }
}