pipeline {
    agent any

    environment {
        DOCKERHUB_USER  = "vanshikak10"
        IMAGE_NAME      = "${DOCKERHUB_USER}/my-cicd-app"
        IMAGE_TAG       = "${BUILD_NUMBER}"
        SONAR_PROJECT   = "my-cicd-app"
    }

    stages {

        stage("Checkout") {
            steps {
                git branch: "main",
                    url: "https://github.com/vanshikakumar10/cicd-pipeline-project.git"
                echo "Code checked out successfully"
            }
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv("sonarqube") {
                    sh """
                        sonar-scanner \
                          -Dsonar.projectKey=${SONAR_PROJECT} \
                          -Dsonar.sources=./app \
                          -Dsonar.host.url=http://sonarqube:9000
                    """
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 5, unit: "MINUTES") {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Build Docker Image") {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ./app"
                sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest"
            }
        }

        stage("Trivy Security Scan") {
            steps {
                sh """
                    docker run aquasec/trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 0 \
                      --format table \
                      ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage("Push to Docker Hub") {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "dockerhub-credentials",
                    usernameVariable: "DOCKER_USER",
                    passwordVariable: "DOCKER_PASS"
                )]) {
                    sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker push ${IMAGE_NAME}:latest"
                }
            }
        }

        stage("Deploy to Kubernetes") {
            steps {
                sh """
                    sed -i '' "s|image:.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|" k8s/deployment.yaml
                    kubectl apply -f k8s/deployment.yaml
                    kubectl apply -f k8s/service.yaml
                    kubectl rollout status deployment/my-cicd-app --timeout=120s
                """
            }
        }
    }

    post {
        success {
            echo "Pipeline SUCCESS! Version ${IMAGE_TAG} deployed."
        }
        failure {
            echo "Pipeline FAILED. Check logs above."
        }
        always {
            sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
        }
    }
}
