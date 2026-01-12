pipeline {
    agent any

    environment {
        IMAGE_NAME     = "myapi-img"
        IMAGE_TAG      = "v1"
        CONTAINER_NAME = "myapi-container"
        NETWORK_NAME   = "jenkins-net"
        SERVICE_PORT   = "8290"

        // WSO2 API Manager info
        WSO2_AM_URL = "https://localhost:9443"
        WSO2_TOKEN = credentials('wso2-api-token')
        API_JSON   = "api-definition.json"
    }

    stages {
        stage('Checkout SCM') {
            steps {
                echo "Cloning repository"
                checkout scm
            }
        }

        stage('Create Docker Network') {
            steps {
                sh """
                docker network inspect ${NETWORK_NAME} >/dev/null 2>&1 || \
                docker network create ${NETWORK_NAME}
                """
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Stop & Remove Old Container') {
            steps {
                sh """
                docker stop ${CONTAINER_NAME} >/dev/null 2>&1 || true
                docker rm ${CONTAINER_NAME} >/dev/null 2>&1 || true
                """
            }
        }

        stage('Run API Container') {
            steps {
                sh """
                docker run -d \
                    --name ${CONTAINER_NAME} \
                    --network ${NETWORK_NAME} \
                    -p ${SERVICE_PORT}:${SERVICE_PORT} \
                    ${IMAGE_NAME}:${IMAGE_TAG}
                """
                echo "Waiting 40 seconds for service to start"
                sleep 40
            }
        }

        stage('Verify API Health') {
            steps {
                script {
                    def apis = [
                        [method: 'GET', path: '/appointmentservices/getAppointment'],
                        [method: 'PUT', path: '/appointmentservices/setAppointment']
                    ]

                    apis.each { api ->
                        echo "Checking API: ${api.method} ${api.path}"
                        def ready = false

                        for (int i = 1; i <= 10; i++) {
                            sleep 10
                            def status = sh(
                                script: "curl -o /dev/null -s -w '%{http_code}' -X ${api.method} http://${CONTAINER_NAME}:${SERVICE_PORT}${api.path}",
                                returnStdout: true
                            ).trim()

                            echo "Attempt ${i}: HTTP ${status}"

                            if (status == "200" || status == "202") {
                                ready = true
                                echo "✔ API ready: ${api.method} ${api.path}"
                                break
                            }
                        }

                        if (!ready) {
                            error "API FAILED: ${api.method} ${api.path} not ready"
                        }
                    }
                }
            }
        }

        stage('Register API in WSO2') {
            steps {
                script {
                    echo "Registering API in WSO2 API Manager"
                    retry(3) {
                        sh """
                        curl -k -X POST ${WSO2_AM_URL}/api/am/publisher/v4/apis \\
                            -H "Authorization: Bearer ${WSO2_TOKEN}" \\
                            -H "Content-Type: application/json" \\
                            -d @${API_JSON}
                        """
                    }
                }
            }
        }

        stage('Smoke Test via API Gateway') {
            steps {
                script {
                    echo "Running smoke test via API Gateway"
                    def paths = [
                        '/appointmentservices/getAppointment',
                        '/appointmentservices/setAppointment'
                    ]
                    paths.each { path ->
                        def code = sh(
                            script: "curl -o /dev/null -s -w '%{http_code}' http://localhost:8243${path}",
                            returnStdout: true
                        ).trim()
                        echo "HTTP ${code} for ${path}"
                        if (code != "200" && code != "202") {
                            error "Smoke test failed for ${path}"
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline completed. Stopping and removing container, cleaning workspace"
            sh """
            docker stop ${CONTAINER_NAME} >/dev/null 2>&1 || true
            docker rm ${CONTAINER_NAME} >/dev/null 2>&1 || true
            """
            cleanWs()
        }
    }
}
