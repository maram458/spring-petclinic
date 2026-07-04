pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "maramaroua/spring-petclinic"
        DOCKER_TAG   = "${BUILD_NUMBER}"
    }

    stages {

        stage('📥 Checkout') {
            steps {
                checkout scm
                echo "✅ Code checked out"
            }
        }

        stage('🧪 Tests') {
            steps {
                sh 'mvn test -Dspring.profiles.active=test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('📦 Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('🐳 Docker Build') {
            steps {
                sh """
                    docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                    docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest
                """
            }
        }

        stage('🚀 Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

        stage('☸️ Deploy to Kubernetes') {
            steps {
                sh """
                    kubectl set image deployment/petclinic \
                        petclinic=${DOCKER_IMAGE}:${DOCKER_TAG}
                    kubectl rollout status deployment/petclinic
                """
            }
        }

        stage('✅ Health Check') {
            steps {
                sh """
                    sleep 10
                    kubectl get pods -l app=petclinic
                """
            }
        }
    }

    post {
        success {
            echo '🎉 Deployed successfully!'
        }
        failure {
            echo '❌ Pipeline failed!'
        }
        always {
            cleanWs()
        }
    }
}
