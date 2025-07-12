pipeline {
    agent any

    environment {
        // Fetches the 'sonar-scanner' tool configured in Jenkins Global Tool Configuration
        SCANNER_HOME = tool 'sonar-scanner'

        // Fetches the SonarCloud token credential by ID.
        // The `withSonarQubeEnv` step will automatically pick this up as SONAR_LOGIN.
        SONAR_AUTH_TOKEN = credentials('sonar-token')

        // Define your correct SonarCloud organization and project keys.
        // These MUST match what's on SonarCloud.
        SONAR_ORGANIZATION = 'ashwin2701'
        SONAR_PROJECT_KEY = 'ashwin2701_ci-jenkins'
    }

    stages {
        // Stage 1: Checkout the source code from your GitHub repository
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        // Stage 2: Perform static code analysis with SonarCloud
        stage('Code-Analysis') {
            steps {
                // 'SonarCloud' must match the name configured in Jenkins
                // (Manage Jenkins -> Configure System -> SonarQube servers)
                withSonarQubeEnv('SonarCloud') {
                    // Execute the SonarScanner CLI
                    sh """
                        ${env.SCANNER_HOME}/bin/sonar-scanner \\
                          -Dsonar.organization=${env.SONAR_ORGANIZATION} \\
                          -Dsonar.projectKey=${env.SONAR_PROJECT_KEY} \\
                          -Dsonar.sources=.
                        """
                    // sonar.host.url and sonar.login are automatically injected by withSonarQubeEnv
                }
            }
        }

        // Stage 3: Wait for the Quality Gate status from SonarCloud
        stage('Quality Gate Check') {
            steps {
                script {
                    // Add a short sleep to allow SonarCloud analysis to fully transition to 'SUCCESS'
                    // and stabilize before Jenkins starts polling for its status.
                    // This helps mitigate the race condition with very fast analyses.
                    echo 'Sleeping for 5 seconds to allow SonarCloud analysis to fully complete and stabilize...'
                    sleep time: 5, unit: 'SECONDS'
                }
                // Timeout for the Quality Gate check (e.g., 15 minutes)
                // This ensures the pipeline doesn't hang indefinitely if SonarCloud is slow
                // or if there's a problem getting the status.
                timeout(time: 15, unit: 'MINUTES') {
                    // This step polls SonarCloud for the Quality Gate status.
                    // If the Quality Gate fails, the pipeline will be aborted.
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // Stage 4: Build and Push Docker Image to Docker Hub
        stage('Docker Build And Push') {
            steps {
                script {
                    // 'docker-cred' must be the ID of your Docker Hub credential in Jenkins
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-cred') {
                        def buildNumber = env.BUILD_NUMBER ?: '1' // Use Jenkins build number for unique tag
                        // Build the Docker image. Assumes Dockerfile is in the root of the repo.
                        def image = docker.build("pekker123/crud-123:${buildNumber}")
                        image.push() // Push image with build number tag

                        // Also push with 'latest' tag, as your deploy step uses it
                        def latestImage = docker.build("pekker123/crud-123:latest")
                        latestImage.push()
                    }
                }
            }
        }

        // Stage 5: Deploy the Docker image to your EC2 instance
        stage('Deploy To EC2') {
            steps {
                script {
                    // Remove any existing containers of the same image to avoid conflicts
                    // The '|| true' makes sure the step doesn't fail if no containers are found
                    sh 'docker rm -f $(docker ps -aq --filter ancestor=pekker123/crud-123) || true'
                    // Pull the latest image from Docker Hub
                    sh 'docker pull pekker123/crud-123:latest'
                    // Run the new container, mapping port 3000 and giving it a name
                    sh 'docker run -d -p 3000:3000 --name crud-app pekker123/crud-123:latest'
                }
            }
        }
    }
}
