pipeline {
    agent any
    
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKERHUB_USERNAME = 'jossef2010'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Docker Images') {
            steps {
                script {
                    // Movie Service bauen
                    dir('movie-service') {
                        sh 'docker build -t jossef2010/movie-service:${BUILD_NUMBER} .'
                    }
                    // Cast Service bauen
                    dir('cast-service') {
                        sh 'docker build -t jossef2010/cast-service:${BUILD_NUMBER} .'
                    }
                }
            }
        }
        
        stage('Test with Docker Compose') {
            steps {
                script {
                    try {
                        sh '''
                    	    echo "=== Starting docker-compose without Nginx ==="
                    	    docker-compose -f docker-compose.yml up -d
                    
                    	    echo "=== Waiting for services to be ready (20 seconds) ==="
                    	    sleep 20

                            echo "=== Listing all running containers ==="
                            docker ps
                    
                            echo "=== Testing Movie Service via container name ==="
                            docker exec movie-cast-multibranch_develop-movie_service-1 curl -f http://localhost:8000/docs || exit 1
                    
                            echo "=== Testing Cast Service via container name ==="
                            docker exec movie-cast-multibranch_develop-cast_service-1 curl -f http://localhost:8000/docs || exit 1
                    
                    	    
                   	    echo "=== All tests passed! ==="
                        '''               
                    } finally {
                        sh 'docker-compose -f docker-compose.yml down'
                    }
                }
            }
        }
        
        stage('Push to DockerHub') {
            when {
                branch 'develop'
            }
            steps {
                script {
                    docker.withRegistry('', 'dockerhub-credentials') {
                        sh """
                            docker tag jossef2010/movie-service:${BUILD_NUMBER} jossef2010/movie-service:dev-${BUILD_NUMBER}
                            docker tag jossef2010/cast-service:${BUILD_NUMBER} jossef2010/cast-service:dev-${BUILD_NUMBER}
                            docker push jossef2010/movie-service:dev-${BUILD_NUMBER}
                            docker push jossef2010/cast-service:dev-${BUILD_NUMBER}
                        """
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            when {
                branch 'develop'
            }
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
    }
    
    post {
        always {
            cleanWs()
            sh 'docker system prune -f || true'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
