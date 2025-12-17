pipeline {
    agent any

    tools {
        jdk "java17"
        maven "Maven3.9.11"
    }

    environment {
        SONAR_HOST_URL = "https://v2code.rtwohealthcare.com"
        SONAR_PROJECT_KEY = "test_v2"

        DOCKER_REGISTRY_URL = "v2deploy.rtwohealthcare.com"
        IMAGE_NAME = "test-v2"
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Maven Build + Tests') {
            steps {
                sh '''
                    cd Additions
                    mvn -B clean verify
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            cd Additions
                            mvn sonar:sonar \
                              -Dsonar.projectKey=test_v2 \
                              -Dsonar.projectName=test_v2 \
                              -Dsonar.host.url=$SONAR_HOST_URL \
                              -Dsonar.token=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage('Check Code Coverage (<80% Email Only)') {
            steps {
                script {
                    sleep 15   // wait for Sonar analysis to complete

                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        def coverage = sh(
                            script: """
                            curl -s -u $SONAR_TOKEN: \
                            "$SONAR_HOST_URL/api/measures/component?component=$SONAR_PROJECT_KEY&metricKeys=coverage" \
                            | jq -r '.component.measures[0].value'
                            """,
                            returnStdout: true
                        ).trim()

                        echo "SonarQube Coverage: ${coverage}%"

                        if (coverage.toFloat() < 80) {
                            echo "Coverage below 80%. Sending Email..."

                            emailext(
                                subject: "⚠ Code Coverage Alert (<80%) - ${env.JOB_NAME}",
                                mimeType: 'text/html',
                                to: "yourmail@gmail.com",
                                body: """
                                <h3>Code Coverage Alert</h3>
                                <p><b>Project:</b> ${env.JOB_NAME}</p>
                                <p><b>Build:</b> #${env.BUILD_NUMBER}</p>
                                <p><b>Coverage:</b> ${coverage}%</p>
                                <p><b>Required:</b> 80%</p>
                                <p>SonarQube: <a href="${SONAR_HOST_URL}">${SONAR_HOST_URL}</a></p>
                                """
                            )
                        } else {
                            echo "Coverage is OK. No email sent."
                        }
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    cd Additions
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG}
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_REGISTRY_URL}/${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-docker-cred',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh '''
                        echo "$PASS" | docker login ${DOCKER_REGISTRY_URL} -u "$USER" --password-stdin
                        docker push ${DOCKER_REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${DOCKER_REGISTRY_URL}/${IMAGE_NAME}:latest
                        docker logout ${DOCKER_REGISTRY_URL}
                    '''
                }
            }
        }
    }
}
