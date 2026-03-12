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
                    dir('movie-service') {
                        sh 'docker build -t jossef2010/movie-service:${BUILD_NUMBER} .'
                    }
                    dir('cast-service') {
                        sh 'docker build -t jossef2010/cast-service:${BUILD_NUMBER} .'
                    }
                }
            }
        }
        
        // TEST-STAGE ENTFERNT!
        
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
