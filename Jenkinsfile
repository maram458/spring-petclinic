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
                sh '''
                    chmod +x mvnw
                    ./mvnw test \
                        -Dspring.profiles.active=test \
                        -Dexclude="**/PostgresIntegrationTests.java" \
                        -Dsurefire.excludes="**/PostgresIntegrationTests.java"
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }
        stage('📦 Build') {
            steps {
                sh './mvnw clean package -DskipTests'
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


	stage('🔒 Security Scan') {
            steps {
                sh '''
                    /var/jenkins_home/bin/trivy image ${DOCKER_IMAGE}:${DOCKER_TAG} \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        --format table
                '''
            }
        }
        stage('🚀 Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push "$DOCKER_IMAGE:$DOCKER_TAG"
                        docker push "$DOCKER_IMAGE:latest"
                    '''
                }
            }
        }
        stage('☸️ Deploy with Helm') {
            steps {
                sh """
                    /var/jenkins_home/helm upgrade --install petclinic \
                        ./helm/petclinic \
                        --set image.tag=${DOCKER_TAG} \
                        --set image.repository=${DOCKER_IMAGE} \
                        --wait \
                        --timeout 120s
                """
            }
        }
        stage('✅ Health Check') {
            steps {
                sh """
                    sleep 10
                    /var/jenkins_home/kubectl get pods -l app=petclinic
                """
            }
        }
    }
    post {
        success {
            echo '🎉 Deployed successfully with Helm!'
        }
        failure {
            echo '❌ Pipeline failed!'
        }
        always {
            cleanWs()
        }
    }
}
