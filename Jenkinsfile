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
                           echo "=== Starting docker-compose ==="
                           docker-compose -f docker-compose.yml up -d
                    
                           echo "=== Waiting 15 seconds for initialization ==="
                           sleep 15
                    
                           echo "=== ALL CONTAINERS (including exited) ==="
                           docker ps -a
                    
                           echo "=== MOVIE SERVICE LOGS ==="
                           docker-compose -f docker-compose.yml logs movie_service
                    
                           echo "=== CAST SERVICE LOGS ==="
                           docker-compose -f docker-compose.yml logs cast_service
                    
                           echo "=== DATABASE LOGS ==="
                           docker-compose -f docker-compose.yml logs movie_db
                           docker-compose -f docker-compose.yml logs cast_db
                    
                           echo "=== TESTING MOVIE SERVICE (if running) ==="
                           docker-compose -f docker-compose.yml exec -T movie_service curl -f http://localhost:8000/docs || echo "Movie service not responding"
                    
                           echo "=== TESTING CAST SERVICE (if running) ==="
                           docker-compose -f docker-compose.yml exec -T cast_service curl -f http://localhost:8000/docs || echo "Cast service not responding"
                       '''
                    } finally {
                        sh 'docker-compose -f docker-compose.yml down || true'
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
