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

        stage('🐳 Kaniko Build & Push') {
            steps {
                sh """
                    cat <<KANIKOEOF > kaniko-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: kaniko-build-${BUILD_NUMBER}
  namespace: default
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: kaniko
        image: gcr.io/kaniko-project/executor:latest
        args:
        - "--context=git://github.com/maram458/spring-petclinic.git#refs/heads/main"
        - "--destination=${DOCKER_IMAGE}:${DOCKER_TAG}"
        - "--destination=${DOCKER_IMAGE}:latest"
        - "--cache=true"
        - "--cache-ttl=24h"
        volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
      volumes:
      - name: docker-config
        secret:
          secretName: dockerhub-kaniko-secret
          items:
          - key: .dockerconfigjson
            path: config.json
KANIKOEOF
                    /var/jenkins_home/kubectl apply -f kaniko-job.yaml
                    /var/jenkins_home/kubectl wait --for=condition=complete job/kaniko-build-${BUILD_NUMBER} -n default --timeout=900s || \
                      (/var/jenkins_home/kubectl logs job/kaniko-build-${BUILD_NUMBER} -n default; exit 1)
                    /var/jenkins_home/kubectl logs job/kaniko-build-${BUILD_NUMBER} -n default
                    /var/jenkins_home/kubectl delete job/kaniko-build-${BUILD_NUMBER} -n default --ignore-not-found
                """
            }
        }

        stage('🔒 Security Scan') {
            steps {
                sh '''
                    /var/jenkins_home/bin/trivy image ${DOCKER_IMAGE}:${DOCKER_TAG} \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        --format table \
                        --timeout 10m \
                        --scanners vuln \
                        --pkg-types os
                '''
            }
        }

        stage('📝 Update GitOps Manifest') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-app-credentials',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    sh """
                        sed -i 's/tag: ".*"/tag: "${DOCKER_TAG}"/' helm/petclinic/values.yaml

                        git config user.email "jenkins@petclinic.local"
                        git config user.name "Jenkins CI"

                        git add helm/petclinic/values.yaml

                        git commit -m "CI: bump image tag to ${DOCKER_TAG}" || echo "No changes to commit"

                        git push https://\${GIT_USER}:\${GIT_TOKEN}@github.com/maram458/spring-petclinic.git HEAD:main
                    """
                }
            }
        }

        stage('✅ Health Check') {
            steps {
                sh """
                    sleep 30
                    /var/jenkins_home/kubectl get pods -l app=petclinic
                """
            }
        }
    }

    post {
        success {
            echo '🎉 Deployed successfully with GitOps + ArgoCD!'
        }

        failure {
            echo '❌ Pipeline failed!'
        }

        always {
            cleanWs()
        }
    }
}
