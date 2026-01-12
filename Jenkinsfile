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
                echo "🔄 Checking out source code from Git"
                checkout scm
            }
        }

        stage('Create Docker Network') {
            steps {
                script {
                    def networkExists = sh(
                        script: "docker network inspect ${NETWORK_NAME} >/dev/null 2>&1 && echo 'found' || echo 'notfound'",
                        returnStdout: true
                    ).trim()
                    
                    if (networkExists == 'notfound') {
                        echo "Creating Docker network '${NETWORK_NAME}'..."
                        sh "docker network create ${NETWORK_NAME}"
                    } else {
                        echo "Docker network '${NETWORK_NAME}' already exists ✅"
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "📦 Building Docker image: ${IMAGE_NAME}"
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }

        stage('Stop & Remove Old Container') {
            steps {
                script {
                    echo "🛑 Stopping and removing old container if exists..."
                    sh "docker stop ${CONTAINER_NAME} || true"
                    sh "docker rm ${CONTAINER_NAME} || true"
                }
            }
        }

        stage('Run API Container') {
            steps {
                echo "▶️ Running API container: ${CONTAINER_NAME}"
                sh "docker run -d --name ${CONTAINER_NAME} --network ${NETWORK_NAME} -p ${SERVICE_PORT}:${SERVICE_PORT} ${IMAGE_NAME}"
                echo "⏳ Waiting 40 seconds for API to start..."
                sleep 40
            }
        }

        stage('Verify API Health') {
            steps {
                script {
                    echo "🔍 Verifying API health at: http://${CONTAINER_NAME}:${SERVICE_PORT}${API_PATH}"
                    def status = sh(
                        script: "curl -o /dev/null -s -w '%{http_code}' -X GET http://${CONTAINER_NAME}:${SERVICE_PORT}${API_PATH}",
                        returnStdout: true
                    ).trim()
                    echo "API Response HTTP Code: ${status}"
                    if (status != '200') {
                        error "❌ API Health check failed! Status: ${status}"
                    } else {
                        echo "✅ API is healthy"
                    }
                }
            }
        }

        stage('Register API in WSO2') {
            steps {
                echo "📝 Skipping for now — add WSO2 registration scripts here"
            }
        }

        stage('Smoke Test via API Gateway') {
            steps {
                echo "⚡ Skipping for now — add API Gateway smoke tests here"
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up: stopping and removing container, cleaning workspace"
            sh "docker stop ${CONTAINER_NAME} || true"
            sh "docker rm ${CONTAINER_NAME} || true"
            cleanWs()
        }
    }
}
