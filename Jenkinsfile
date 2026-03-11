pipeline {
    agent any
    
    environment {
        // DockerHub credentials (configure these in Jenkins)
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKERHUB_USERNAME = 'jossef2010'
        
        // Service names
        MOVIE_SERVICE = 'movie-service'
        CAST_SERVICE = 'cast-service'
        
        // Image repository
        MOVIE_IMAGE = "${DOCKERHUB_USERNAME}/movie-service"
        CAST_IMAGE = "${DOCKERHUB_USERNAME}/cast-service"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build and Test') {
            parallel {
                stage('Build Movie Service') {
                    steps {
                        sh """
                            cd ${MOVIE_SERVICE}
                            docker build -t ${MOVIE_IMAGE}:${BUILD_NUMBER} .
                        """
                    }
                }
                
                stage('Build Cast Service') {
                    steps {
                        sh """
                            cd ${CAST_SERVICE}
                            docker build -t ${CAST_IMAGE}:${BUILD_NUMBER} .
                        """
                    }
                }
                
                stage('Run Tests') {
                    steps {
                        sh """
                            docker-compose -f docker-compose.yml up -d
                            sleep 10
                            # Add your test commands here
                            curl -f http://localhost:8080/api/v1/movies/docs || exit 1
                            curl -f http://localhost:8080/api/v1/casts/docs || exit 1
                            docker-compose -f docker-compose.yml down
                        """
                    }
                }
            }
        }
        
        stage('Push Docker Images') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'qa'
                    branch 'staging'
                    branch 'main'
                }
            }
            stages {
                stage('Push to Dev') {
                    when { branch 'develop' }
                    steps {
                        script {
                            docker.withRegistry('', DOCKERHUB_CREDENTIALS) {
                                sh """
                                    docker tag ${MOVIE_IMAGE}:${BUILD_NUMBER} ${MOVIE_IMAGE}:dev-${BUILD_NUMBER}
                                    docker tag ${CAST_IMAGE}:${BUILD_NUMBER} ${CAST_IMAGE}:dev-${BUILD_NUMBER}
                                    docker push ${MOVIE_IMAGE}:dev-${BUILD_NUMBER}
                                    docker push ${CAST_IMAGE}:dev-${BUILD_NUMBER}
                                """
                            }
                        }
                    }
                }
                
                stage('Push to QA') {
                    when { branch 'qa' }
                    steps {
                        script {
                            docker.withRegistry('', DOCKERHUB_CREDENTIALS) {
                                sh """
                                    docker tag ${MOVIE_IMAGE}:${BUILD_NUMBER} ${MOVIE_IMAGE}:qa-${BUILD_NUMBER}
                                    docker tag ${CAST_IMAGE}:${BUILD_NUMBER} ${CAST_IMAGE}:qa-${BUILD_NUMBER}
                                    docker push ${MOVIE_IMAGE}:qa-${BUILD_NUMBER}
                                    docker push ${CAST_IMAGE}:qa-${BUILD_NUMBER}
                                """
                            }
                        }
                    }
                }
                
                stage('Push to Staging') {
                    when { branch 'staging' }
                    steps {
                        script {
                            docker.withRegistry('', DOCKERHUB_CREDENTIALS) {
                                sh """
                                    docker tag ${MOVIE_IMAGE}:${BUILD_NUMBER} ${MOVIE_IMAGE}:staging-${BUILD_NUMBER}
                                    docker tag ${CAST_IMAGE}:${BUILD_NUMBER} ${CAST_IMAGE}:staging-${BUILD_NUMBER}
                                    docker push ${MOVIE_IMAGE}:staging-${BUILD_NUMBER}
                                    docker push ${CAST_IMAGE}:staging-${BUILD_NUMBER}
                                """
                            }
                        }
                    }
                }
                
                stage('Push to Production') {
                    when { branch 'main' }
                    steps {
                        script {
                            docker.withRegistry('', DOCKERHUB_CREDENTIALS) {
                                sh """
                                    docker tag ${MOVIE_IMAGE}:${BUILD_NUMBER} ${MOVIE_IMAGE}:prod-${BUILD_NUMBER}
                                    docker tag ${MOVIE_IMAGE}:${BUILD_NUMBER} ${MOVIE_IMAGE}:latest
                                    docker tag ${CAST_IMAGE}:${BUILD_NUMBER} ${CAST_IMAGE}:prod-${BUILD_NUMBER}
                                    docker tag ${CAST_IMAGE}:${BUILD_NUMBER} ${CAST_IMAGE}:latest
                                    docker push ${MOVIE_IMAGE}:prod-${BUILD_NUMBER}
                                    docker push ${MOVIE_IMAGE}:latest
                                    docker push ${CAST_IMAGE}:prod-${BUILD_NUMBER}
                                    docker push ${CAST_IMAGE}:latest
                                """
                            }
                        }
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            stages {
                stage('Deploy to Dev') {
                    when { branch 'develop' }
                    steps {
                        sh """
                            helm upgrade --install movie-service ./charts \
                                --namespace dev \
                                --set movie-service.image.tag=dev-${BUILD_NUMBER} \
                                --set cast-service.image.tag=dev-${BUILD_NUMBER} \
                                --set environment=dev
                        """
                    }
                }
                
                stage('Deploy to QA') {
                    when { branch 'qa' }
                    steps {
                        input message: 'Approve deployment to QA?', ok: 'Deploy'
                        sh """
                            helm upgrade --install movie-service ./charts \
                                --namespace qa \
                                --set movie-service.image.tag=qa-${BUILD_NUMBER} \
                                --set cast-service.image.tag=qa-${BUILD_NUMBER} \
                                --set environment=qa
                        """
                    }
                }
                
                stage('Deploy to Staging') {
                    when { branch 'staging' }
                    steps {
                        input message: 'Approve deployment to Staging?', ok: 'Deploy'
                        sh """
                            helm upgrade --install movie-service ./charts \
                                --namespace staging \
                                --set movie-service.image.tag=staging-${BUILD_NUMBER} \
                                --set cast-service.image.tag=staging-${BUILD_NUMBER} \
                                --set environment=staging
                        """
                    }
                }
                
                stage('Deploy to Production') {
                    when { branch 'main' }
                    steps {
                        input message: 'APPROVE PRODUCTION DEPLOYMENT?', ok: 'Deploy to Production'
                        sh """
                            helm upgrade --install movie-service ./charts \
                                --namespace prod \
                                --set movie-service.image.tag=prod-${BUILD_NUMBER} \
                                --set cast-service.image.tag=prod-${BUILD_NUMBER} \
                                --set environment=prod \
                                --set replicaCount=3
                        """
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            stages {
                stage('Verify Dev') {
                    when { branch 'develop' }
                    steps {
                        sh """
                            kubectl wait --namespace=dev --for=condition=ready pod -l app=movie-service --timeout=60s
                            kubectl port-forward -n dev service/movie-service 8081:8080 &
                            sleep 5
                            curl -f http://localhost:8081/api/v1/movies/docs || exit 1
                            curl -f http://localhost:8081/api/v1/casts/docs || exit 1
                            kill %1
                        """
                    }
                }
                
                stage('Verify QA') {
                    when { branch 'qa' }
                    steps {
                        sh """
                            kubectl wait --namespace=qa --for=condition=ready pod -l app=movie-service --timeout=60s
                            kubectl port-forward -n qa service/movie-service 8082:8080 &
                            sleep 5
                            curl -f http://localhost:8082/api/v1/movies/docs || exit 1
                            curl -f http://localhost:8082/api/v1/casts/docs || exit 1
                            kill %1
                        """
                    }
                }
                
                stage('Verify Staging') {
                    when { branch 'staging' }
                    steps {
                        sh """
                            kubectl wait --namespace=staging --for=condition=ready pod -l app=movie-service --timeout=60s
                            kubectl port-forward -n staging service/movie-service 8083:8080 &
                            sleep 5
                            curl -f http://localhost:8083/api/v1/movies/docs || exit 1
                            curl -f http://localhost:8083/api/v1/casts/docs || exit 1
                            kill %1
                        """
                    }
                }
                
                stage('Verify Production') {
                    when { branch 'main' }
                    steps {
                        sh """
                            kubectl wait --namespace=prod --for=condition=ready pod -l app=movie-service --timeout=60s
                            kubectl port-forward -n prod service/movie-service 8084:8080 &
                            sleep 5
                            curl -f http://localhost:8084/api/v1/movies/docs || exit 1
                            curl -f http://localhost:8084/api/v1/casts/docs || exit 1
                            kill %1
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
            sh 'docker system prune -f'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
