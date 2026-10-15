pipeline {
    agent any

    environment {
        DOCKER_HUB_USER = 'tanshul12'
        IMAGE_NAME      = 'online-bookstore'
        IMAGE_TAG       = "${BUILD_NUMBER}"
        KUBECONFIG_CRED = 'kuberconfig'
        DOCKER_CRED     = 'dockerhub'
        DOCKER_BIN      = 'C:\\Users\\tanshul\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin'
    }

    stages {
        stage('1. Clone Code') {
            steps {
                checkout scm
            }
        }

        stage('2. Build Docker Image') {
            steps {
                bat "\"${DOCKER_BIN}\\docker.exe\" build -t %DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG% ."
                bat "\"${DOCKER_BIN}\\docker.exe\" tag %DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG% %DOCKER_HUB_USER%/%IMAGE_NAME%:latest"
            }
        }

        stage('3. Automated Testing & Quality Gate') {
            steps {
                bat """
                    "${DOCKER_BIN}\\docker.exe" run -d --name test-bookstore -p 8086:80 %DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG%
                    ping -n 4 127.0.0.1 >nul
                    curl.exe -s -f http://localhost:8086 >nul || ("${DOCKER_BIN}\\docker.exe" rm -f test-bookstore && exit 1)
                    "${DOCKER_BIN}\\docker.exe" rm -f test-bookstore
                """
            }
        }

        stage('4. Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: DOCKER_CRED, usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    bat "\"${DOCKER_BIN}\\docker.exe\" login -u %DH_USER% -p %DH_PASS%"
                    bat "\"${DOCKER_BIN}\\docker.exe\" push %DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG%"
                    bat "\"${DOCKER_BIN}\\docker.exe\" push %DOCKER_HUB_USER%/%IMAGE_NAME%:latest"
                }
            }
        }

        stage('5. Kubernetes Deploy & Rollout Verification Gate') {
            steps {
                withCredentials([file(credentialsId: KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
                    script {
                        bat "\"${DOCKER_BIN}\\kubectl.exe\" --kubeconfig=\"%KUBECONFIG_FILE%\" apply -f deployment.yaml"
                        bat "\"${DOCKER_BIN}\\kubectl.exe\" --kubeconfig=\"%KUBECONFIG_FILE%\" set image deployment/bookstore-deployment bookstore=%DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG%"

                        def status = bat(
                            script: "\"${DOCKER_BIN}\\kubectl.exe\" --kubeconfig=\"%KUBECONFIG_FILE%\" rollout status deployment/bookstore-deployment --timeout=60s",
                            returnStatus: true
                        )

                        if (status != 0) {
                            echo "Deployment gate failed! Initiating automated rollback to previous stable release..."
                            bat "\"${DOCKER_BIN}\\kubectl.exe\" --kubeconfig=\"%KUBECONFIG_FILE%\" rollout undo deployment/bookstore-deployment"
                            bat "\"${DOCKER_BIN}\\kubectl.exe\" --kubeconfig=\"%KUBECONFIG_FILE%\" rollout status deployment/bookstore-deployment"
                            error("Rollout Gate Failed: Defective release rolled back.")
                        } else {
                            echo "Deployment Gate Passed: 3 replicas are active and healthy."
                        }
                    }
                }
            }
        }
    }
}
