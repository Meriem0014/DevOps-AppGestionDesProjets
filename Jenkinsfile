
pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'meriem0014'
        IMAGE_NAME = 'backend-app'
        IMAGE_TAG = 'latest'
        DOCKER_NETWORK = 'backend-network'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Meriem0014/DevOps-AppGestionDesProjets.git'
            }
        }

        stage('Build Maven') {
            steps {
                dir('backend') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Docker Build') {
            steps {
                dir('backend') {
                    sh '''
                        docker build \
                          -t backend-app:latest \
                          -t meriem0014/backend-app:latest .
                    '''
                }
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_TOKEN" | docker login \
                          -u "$DOCKER_USER" \
                          --password-stdin

                        docker push meriem0014/backend-app:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy MySQL') {
            steps {
                sh '''
                    docker network create backend-network || true
                    docker volume create mysql_data

                    if ! docker ps -a --format '{{.Names}}' | grep -qx mysql; then
                        docker run -d \
                          --name mysql \
                          --network backend-network \
                          -p 3306:3306 \
                          -v mysql_data:/var/lib/mysql \
                          -e MYSQL_ROOT_PASSWORD=root \
                          -e MYSQL_DATABASE=test_db \
                          --health-cmd="mysqladmin ping -h 127.0.0.1 -uroot -proot" \
                          --health-interval=10s \
                          --health-timeout=5s \
                          --health-retries=10 \
                          mysql:8.0
                    else
                        docker start mysql || true
                        docker network connect backend-network mysql || true
                    fi

                    echo "Waiting for MySQL..."

                    for i in $(seq 1 30); do
                        if docker exec mysql mysqladmin ping \
                          -h 127.0.0.1 -uroot -proot --silent; then
                            echo "MySQL is ready!"
                            exit 0
                        fi
                        sleep 5
                    done

                    echo "MySQL did not become ready."
                    exit 1
                '''
            }
        }

        stage('Deploy Backend') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_TOKEN" | docker login \
                          -u "$DOCKER_USER" \
                          --password-stdin

                        docker pull meriem0014/backend-app:latest

                        docker rm -f backend-app || true

                        docker run -d \
                          --name backend-app \
                          --network backend-network \
                          --restart unless-stopped \
                          -p 8080:8080 \
                          -e SPRING_DATASOURCE_URL="jdbc:mysql://mysql:3306/test_db?createDatabaseIfNotExist=true" \
                          -e SPRING_DATASOURCE_USERNAME=root \
                          -e SPRING_DATASOURCE_PASSWORD=root \
                          meriem0014/backend-app:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Running containers:"
                    docker ps

                    echo "Backend logs:"
                    sleep 10
                    docker logs --tail 50 backend-app
                '''
            }
        }
    }

    post {
        success {
            echo 'CD Pipeline completed successfully!'
        }

        failure {
            echo 'CD Pipeline failed. Check the stage logs.'
        }
    }
}

