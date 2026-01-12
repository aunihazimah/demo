pipeline {
    agent any

    environment {
        CONTAINER_NAME = "myapi-container"
        IMAGE_NAME = "myapi-img:v1"
        NETWORK_NAME = "jenkins-net"
        SERVICE_PORT = "8290"
        API_PATH = "/appointmentservices/getAppointment"
        WSO2_TOKEN = credentials('wso2-api-token')
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Create Docker Network') {
            steps {
                script {
                    def networkExists = sh(
                        script: "docker network inspect ${NETWORK_NAME} >/dev/null 2>&1 || echo 'notfound'",
                        returnStdout: true
                    ).trim()
                    
                    if (networkExists == 'notfound') {
                        echo "Creating Docker network '${NETWORK_NAME}'..."
                        sh "docker network create ${NETWORK_NAME}"
                    } else {
                        echo "Docker network '${NETWORK_NAME}' already exists"
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image: ${IMAGE_NAME}"
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }

        stage('Stop & Remove Old Container') {
            steps {
                script {
                    echo "Stopping and removing old container if exists..."
                    sh "docker stop ${CONTAINER_NAME} || true"
                    sh "docker rm ${CONTAINER_NAME} || true"
                }
            }
        }

        stage('Run API Container') {
            steps {
                sh "docker run -d --name ${CONTAINER_NAME} --network ${NETWORK_NAME} -p ${SERVICE_PORT}:${SERVICE_PORT} ${IMAGE_NAME}"
                echo "⏳ Waiting 40 seconds for service to start..."
                sleep 40
            }
        }

        stage('Verify API Health') {
            steps {
                script {
                    echo "Checking API: GET ${API_PATH}"
                    // Use localhost instead of container name
                    def status = sh(
                        script: "curl -o /dev/null -s -w '%{http_code}' -X GET http://localhost:${SERVICE_PORT}${API_PATH}",
                        returnStdout: true
                    ).trim()
                    echo "API Response HTTP Code: ${status}"
                    if (status != '200') {
                        error "API Health check failed! Status: ${status}"
                    }
                }
            }
        }

        stage('Register API in WSO2') {
            steps {
                echo "Skipping for now (add WSO2 registration scripts here)"
            }
        }

        stage('Smoke Test via API Gateway') {
            steps {
                echo "Skipping for now (add API Gateway smoke test scripts here)"
            }
        }
    }

    post {
        always {
            echo "Cleaning up: stopping and removing container, cleaning workspace"
            sh "docker stop ${CONTAINER_NAME} || true"
            sh "docker rm ${CONTAINER_NAME} || true"
            cleanWs()
        }
    }
}
