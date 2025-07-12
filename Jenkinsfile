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

        stage('Code-Analysis') {
            steps {
                withSonarQubeEnv('SonarCloud') {
                    sh """
                        ${env.SCANNER_HOME}/bin/sonar-scanner \\
                          -Dsonar.organization=${env.SONAR_ORGANIZATION} \\
                          -Dsonar.projectKey=${env.SONAR_PROJECT_KEY} \\
                          -Dsonar.sources=.
                        """
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                script {
                    echo 'Sleeping for 5 seconds to allow SonarCloud analysis to fully complete and stabilize...'
                    sleep time: 5, unit: 'SECONDS'
                }
                timeout(time: 15, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build And Push') {
            steps {
                script {
                    // --- TEMPORARY DEBUGGING STEP FOR DOCKER CREDENTIALS ---
                    // This block explicitly fetches credentials into variables
                    // and attempts a manual docker login to test resolution.
                    withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        echo "Attempting manual Docker login for debugging..."
                        // The 'echo' will show '****' for masked secrets, but the command will use the actual value.
                        sh """
                            echo "DOCKER_USERNAME retrieved: ${DOCKER_USERNAME}"
                            echo "DOCKER_PASSWORD retrieved: ${DOCKER_PASSWORD}"
                            echo "${DOCKER_PASSWORD}" | docker login -u "${DOCKER_USERNAME}" --password-stdin https://index.docker.io/v1/
                        """
                        echo "Manual Docker login command executed. Check console output for success/failure."
                    }
                    // --- END OF TEMPORARY DEBUGGING STEP ---


                    // Now, proceed with Docker operations using the standard method.
                    // If the manual login above works, but the one below fails,
                    // it points to an issue with 'docker.withRegistry' itself.
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-cred') {
                        def buildNumber = env.BUILD_NUMBER ?: '1'
                        def image = docker.build("ash27win/crud-123:${buildNumber}")
                        image.push()

                        // Pushing the 'latest' tag
                        def latestImage = docker.build("ash27win/crud-123:latest")
                        latestImage.push()
                    }
                }
            }
        }

        stage('Deploy To EC2') {
            steps {
                script {
                    sh 'docker stop crud-app || true'
                    sh 'docker rm crud-app || true'
                    sh 'docker pull ash27win/crud-123:latest'
                    sh 'docker run -d -p 3000:3000 --name crud-app ash27win/crud-123:latest'
                }
            }
        }
    }
}
