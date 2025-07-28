@Library("Shared") _
pipeline {
    agent { label 'kali' } // or replace with the appropriate agent label

    environment {
        DB_NAME = 'test_db'
        DB_USER = 'root'
        DB_PASSWORD = 'root'
        DB_PORT = '3306'
        DB_HOST = 'db_cont'
        NETWORK_NAME = 'notes_net'
        MYSQL_CONTAINER = 'db_cont'
        APP_CONTAINER = 'notes_app'
        IMAGE_NAME = 'notes-app:latest'
    }

    stages {
        stage("hello"){
            steps{
                script{
                    hello()
                }
            }
        }    
        
        stage('Clone Repository') {
            steps {
                
                script{
                    clone('https://github.com/MUGHEESULHASSAN/django_notes_app.git','main')
                    //git branch: 'main', url: 'https://github.com/MUGHEESULHASSAN/django_notes_app.git'
                }
            }
        }

        stage('Create Docker Network') {
            steps {
                echo '🔗 Creating Docker network (if not exists)...'
                sh "docker network ls | grep ${NETWORK_NAME} || docker network create ${NETWORK_NAME}"
            }
        }

        stage('Start MySQL Container') {
            steps {
                echo '🐬 Starting MySQL container...'
                sh '''
                    docker ps -a --format '{{.Names}}' | grep -q "^${MYSQL_CONTAINER}$" || \
                    docker run -d \
                        --name ${MYSQL_CONTAINER} \
                        --network ${NETWORK_NAME} \
                        -e MYSQL_ROOT_PASSWORD=${DB_PASSWORD} \
                        -e MYSQL_DATABASE=${DB_NAME} \
                        -p ${DB_PORT}:3306 \
                        mysql:5.7
                '''
            }
        }

stage('Wait for MySQL to be Ready') {
    steps {
        echo '⏳ Waiting for MySQL service to be ready...'
        sh '''
            for i in {1..10}; do
              docker exec ${MYSQL_CONTAINER} mysqladmin ping -h"localhost" -uroot -p${DB_PASSWORD} --silent && break
              echo "Waiting for MySQL to be ready..."
              sleep 5
            done
        '''
    }
}


        stage('Build Django Image') {
            steps {
                echo '🐳 Building Django Docker image...'
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }
stage('Push to DockerHub') {
    steps {
        echo '📤 Pushing the image to DockerHub...'
        withCredentials([usernamePassword(credentialsId: 'dockerhubcred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            sh '''
                echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                docker tag notes-app:latest $DOCKER_USER/notes-app:latest
                docker push $DOCKER_USER/notes-app:latest
            '''
        }
    }
}


        stage('Run Django Container') {
            steps {
                echo '🚀 Running Django app container...'
                sh '''
                    docker rm -f ${APP_CONTAINER} || true

                    docker run -d \
                        --name ${APP_CONTAINER} \
                        --network ${NETWORK_NAME} \
                        -e DB_NAME=${DB_NAME} \
                        -e DB_USER=${DB_USER} \
                        -e DB_PASSWORD=${DB_PASSWORD} \
                        -e DB_PORT=${DB_PORT} \
                        -e DB_HOST=${DB_HOST} \
                        -p 8000:8000 \
                        ${IMAGE_NAME} \
                        sh -c "sleep 10 && python manage.py runserver 0.0.0.0:8000"
                '''
            }
        }

        stage('Apply Migrations') {
            steps {
                echo '🛠️ Applying Django migrations...'
                sh '''
                    docker exec ${APP_CONTAINER} python manage.py makemigrations
                    docker exec ${APP_CONTAINER} python manage.py migrate
                '''
            }
        }

        stage('Verify App is Running') {
            steps {
                echo '🔍 Verifying deployment...'
                sh 'curl -I http://localhost:8000 || true'
            }
        }
    }

    post {
        success {
            echo '✅ Deployment successful!'
        }
        failure {
            echo '❌ Deployment failed. Check logs.'
        }
    }
}
