pipeline {
    agent any

    environment {
        APP_NAME = 'java-app'
        IMAGE_NAME = 'moncadar/java-app:latest'
        SONAR_PROJECT_KEY = 'java-app'
        SONAR_HOST_URL = 'http://sonar:9000'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            steps {
                script {
                    docker.image('maven:3.9.6-eclipse-temurin-17').inside('--user root') {
                        sh 'mvn clean test package'
                    }
                }
            }
        }

        stage('Analyze Code Quality') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        script {
                            docker.image('maven:3.9.6-eclipse-temurin-17').inside('--user root --network ci_network') {
                                sh '''
                                    mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.5.0.6356:sonar \
                                      -Dsonar.projectKey=$SONAR_PROJECT_KEY \
                                      -Dsonar.host.url=$SONAR_HOST_URL \
                                      -Dsonar.token=$SONAR_TOKEN
                                '''
                            }
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME .'
            }
        }

        stage('Push to Docker Registry') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push $IMAGE_NAME
                    '''
                }
            }
        }


       stage('Deploy to Kubernetes') {
    steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
            sh '''
                set -e

                echo "Available contexts:"
                kubectl config get-contexts

                echo "Switching to docker-desktop..."
                kubectl config use-context docker-desktop

                echo "Verifying cluster..."
                kubectl config current-context
                kubectl get nodes

                echo "Applying deployment..."
                kubectl apply -f deployment.yaml

                echo "Checking deployment..."
                kubectl get deployments
                kubectl get pods -o wide
            '''
        }
    }
}
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed.'
        }
        always {
            cleanWs()
        }
    }
}
