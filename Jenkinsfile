pipeline {
    agent any

    tools {
        jdk "java17"
        maven "Maven3.9.11"
    }

    environment {
        SONAR_HOST_URL = "https://v2code.rtwohealthcare.com"
        SONAR_TOKEN = "sqp_c845971e99d79e08e5b3024752e6235cd4f41dec"

        // -------- DOCKER (FIXED) --------
        DOCKER_REGISTRY_URL = "v2deploy.rtwohealthcare.com"
        DOCKER_REPO = "test-v2"          // ← Nexus repo name
        REGISTRY_CREDENTIALS = "docker-repo"   // ← existing Jenkins credential

        IMAGE_NAME = "test-v2"
        IMAGE_TAG  = "v${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Maven Build + Tests') {
            steps {
                sh """
                    cd Additions
                    mvn -B clean verify
                """
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        cd Additions
                        mvn sonar:sonar \
                            -Dsonar.projectKey=test_v2 \
                            -Dsonar.projectName=test_v2 \
                            -Dsonar.host.url=${SONAR_HOST_URL} \
                            -Dsonar.token=${SONAR_TOKEN}
                    """
                }
            }
        }

        // ❌ QUALITY GATE REMOVED (unchanged)

        /* ------------ DOCKER FIXES START ------------ */

        stage('Docker Build') {
            steps {
                sh """
                    cd Additions
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .

                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} \
                      ${DOCKER_REGISTRY_URL}/${DOCKER_REPO}/${IMAGE_NAME}:${IMAGE_TAG}

                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} \
                      ${DOCKER_REGISTRY_URL}/${DOCKER_REPO}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${REGISTRY_CREDENTIALS}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo "$DOCKER_PASS" | docker login ${DOCKER_REGISTRY_URL} \
                          -u "$DOCKER_USER" --password-stdin

                        docker push ${DOCKER_REGISTRY_URL}/${DOCKER_REPO}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${DOCKER_REGISTRY_URL}/${DOCKER_REPO}/${IMAGE_NAME}:latest

                        docker logout ${DOCKER_REGISTRY_URL}
                    """
                }
            }
        }

        /* ------------ DOCKER FIXES END ------------ */
    }
}
