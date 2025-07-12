pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        SONAR_AUTH_TOKEN = credentials('sonar-token')
        SONAR_ORGANIZATION = 'ashwin2701'
        SONAR_PROJECT_KEY = 'ashwin2701_ci-jenkins'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Code Analysis') {
            steps {
                withSonarQubeEnv('SonarCloud') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \\
                        -Dsonar.organization=${SONAR_ORGANIZATION} \\
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \\
                        -Dsonar.sources=.
                    """
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                script {
                    def reportFile = '.scannerwork/report-task.txt'
                    if (!fileExists(reportFile)) {
                        error "Sonar report-task.txt not found!"
                    }

                    def props = readProperties file: reportFile
                    def taskId = props['ceTaskId']
                    if (!taskId) {
                        error "ceTaskId not found in report-task.txt"
                    }

                    def maxAttempts = 30
                    def sleepSeconds = 10
                    def analysisId = null

                    for (int i = 0; i < maxAttempts; i++) {
                        echo "Polling SonarCloud for task status (attempt ${i + 1}/${maxAttempts})..."
                        def response = sh(script: "curl -s https://sonarcloud.io/api/ce/task?id=${taskId}", returnStdout: true).trim()
                        def json = new groovy.json.JsonSlurperClassic().parseText(response)

                        if (json?.task?.status == 'SUCCESS') {
                            analysisId = json?.task?.analysisId
                            if (analysisId) {
                                echo "SonarCloud task completed successfully. Analysis ID: ${analysisId}"
                                break
                            } else {
                                echo "Status is SUCCESS but analysisId not yet available. Retrying..."
                            }
                        } else if (json?.task?.status == 'FAILED') {
                            error "Sonar analysis failed."
                        } else {
                            echo "Current status: ${json?.task?.status}. Waiting..."
                        }
                        sleep sleepSeconds
                    }

                    if (!analysisId) {
                        error "SonarCloud analysis did not complete in time or analysisId not available."
                    }

                    def qgResponse = sh(script: "curl -s https://sonarcloud.io/api/qualitygates/project_status?analysisId=${analysisId}", returnStdout: true).trim()
                    def qgJson = new groovy.json.JsonSlurperClassic().parseText(qgResponse)
                    def status = qgJson?.projectStatus?.status

                    echo "SonarCloud Quality Gate result: ${status}"
                    if (status != 'OK') {
                        error "Quality Gate failed: ${status}"
                    }
                }
            }
        }

        stage('Docker Build And Push') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-cred') {
                        def buildNumber = env.BUILD_NUMBER ?: '1'
                        def image = docker.build("pekker123/crud-123:${buildNumber}")
                        image.push()

                        def latestImage = docker.build("pekker123/crud-123:latest")
                        latestImage.push()
                    }
                }
            }
        }

        stage('Deploy To EC2') {
            steps {
                script {
                    sh 'docker rm -f $(docker ps -aq --filter ancestor=pekker123/crud-123) || true'
                    sh 'docker pull pekker123/crud-123:latest'
                    sh 'docker run -d -p 3000:3000 --name crud-app pekker123/crud-123:latest'
                }
            }
        }
    }
}
