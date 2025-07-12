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
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-cred') {
                        def buildNumber = env.BUILD_NUMBER ?: '1'
                        def image = docker.build("ash27win/crud-123:${buildNumber}")
                        image.push()

                        docker.withRegistry('https://index.docker.io/v1/', 'docker-cred') {
                            def latestImage = docker.build("ash27win/crud-123:latest")
                            latestImage.push()
                        }
                    }
                }
            }
        }

        stage('Deploy To EC2') {
            steps {
                script {
                    // --- NEW LINES ADDED HERE TO HANDLE CONTAINER CLEANUP ---
                    // Stop the container if it's running (ignore error if not running)
                    sh 'docker stop crud-app || true'
                    // Remove the container by name if it exists (running or exited)
                    sh 'docker rm crud-app || true'
                    // --- END OF NEW LINES ---

                    // The original line for removing containers by ancestor image is still useful for
                    // cleaning up unnamed containers or those from older image versions, but
                    // the two lines above directly address the "name conflict" error.
                    // sh 'docker rm -f $(docker ps -aq --filter ancestor=ash27win/crud-123) || true'


                    sh 'docker pull ash27win/crud-123:latest'
                    sh 'docker run -d -p 3000:3000 --name crud-app ash27win/crud-123:latest'
                }
            }
        }
    }
}
