pipeline {
    agent any

    environment {
        APP_NAME = 'java-app'
        DOCKER_IMAGE = 'moncadar/java-app:latest'
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
                    docker.image('maven:3.9.6-eclipse-temurin-17').inside('--network ci_network') {
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
                            docker.image('maven:3.9.6-eclipse-temurin-17').inside('--network ci_network') {
                                sh """
                                    mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.5.0.6356:sonar \
                                      -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                                      -Dsonar.host.url=${SONAR_HOST_URL} \
                                      -Dsonar.token=${SONAR_TOKEN}
                                """
                            }
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${DOCKER_IMAGE} .'
            }
        }

        stage('Push to Docker Registry') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                        docker push ${DOCKER_IMAGE}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        kubectl config current-context
                        kubectl get nodes
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
