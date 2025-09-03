pipeline {
    agent any

    environment {
        IMAGE_NAME = 'gmail-clone'
        IMAGE_TAG = 'v1'
    }

    stages {
        stage('Checkout Code') {
            steps {
                    checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: '1234', url: 'https://github.com/harikaaws2025/login.git']])
                sh 'pwd'
                sh 'ls -l'
            }
        }

        stage('Build with Maven') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                        docker stop ${IMAGE_NAME}:${IMAGE_TAG} || true
                        docker rmi -f ${IMAGE_NAME}:${IMAGE_TAG} || true
                        docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    """
                }
            }
        }

        stage('Run Docker Container') {
            steps {
               sh '''
    if lsof -i :8080; then
        echo "Port 8080 is busy. Killing conflicting container..."
        docker ps --filter "publish=8080" -q | xargs -r docker rm -f
    fi
    docker run -d --name gmail_container -p 8080:8080 gmail-clone:v1
'''

            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} $DOCKER_USER/${IMAGE_NAME}:${IMAGE_TAG}
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push $DOCKER_USER/${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy to Docker Swarm') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        docker swarm init || true

                        cat <<EOF > docker-stack.yml
version: '3.8'
services:
  gmail-service:
    image: $DOCKER_USER/${IMAGE_NAME}:${IMAGE_TAG}
    ports:
      - "8090:8080"
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
EOF

                        docker stack deploy -c docker-stack.yml gmail_stack
                        docker stack services gmail_stack
                    """
                }
            }
        }
    }
}
