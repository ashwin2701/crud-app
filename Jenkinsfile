pipeline {
    agent any

    environment {
        // Fetch the sonar-scanner tool; it will set SCANNER_HOME automatically
        // If 'sonar-scanner' is the name configured in Manage Jenkins -> Global Tool Configuration
        // This is often implicitly handled by `withSonarQubeEnv` if a `tool` step isn't used before it,
        // but it's good practice to ensure the tool is available.
        // If you define it here, you should then use ${SCANNER_HOME} in your sh command.

        // Fetch the SonarCloud token credential.
        // The `withSonarQubeEnv` step will then pick this up as SONAR_LOGIN.
        SONAR_AUTH_TOKEN = credentials('sonar-token')

        // Define your correct SonarCloud organization and project keys as environment variables
        // This makes them reusable and easier to manage.
        SONAR_ORGANIZATION = 'ashwin2701' // CORRECTED: Use your actual SonarCloud organization
        SONAR_PROJECT_KEY = 'ashwin2701_ci-jenkins' // CORRECTED: Use your actual SonarCloud project key
    }

    stages {
        // CRITICAL: Add the checkout SCM stage to clone your code
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Code-Analysis') {
            steps {
                // 'SonarCloud' must match the name configured in Jenkins (Manage Jenkins -> Configure System -> SonarQube servers)
                withSonarQubeEnv('SonarCloud') {
                    script {
                        // Retrieve the path to the SonarScanner CLI tool
                        // 'sonar-scanner' must match the name configured in Manage Jenkins -> Global Tool Configuration
                        def scannerHome = tool 'sonar-scanner'

                        sh """
                            ${scannerHome}/bin/sonar-scanner \\
                              -Dsonar.organization=${env.SONAR_ORGANIZATION} \\
                              -Dsonar.projectKey=${env.SONAR_PROJECT_KEY} \\
                              -Dsonar.sources=.
                            """
                        // Note: sonar.host.url and sonar.login are automatically injected by withSonarQubeEnv
                        // No need to pass -Dsonar.host.url=https://sonarcloud.io explicitly here.
                        // The backslashes allow the command to span multiple lines.
                    }
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                timeout(time: 5, unit: 'MINUTES') { // Max time to wait for analysis to complete
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build And Push') {
            steps {
                script {
                    // Make sure 'docker-cred' is the correct ID of your Docker Hub credential in Jenkins
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-cred') { // Explicitly define Docker Hub registry URL
                        def buildNumber = env.BUILD_NUMBER ?: '1'
                        // Make sure the Dockerfile is in the root of your checked-out repository
                        def image = docker.build("pekker123/crud-123:${buildNumber}") // Use build number for unique tag
                        image.push()
                        docker.withRegistry('', 'docker-cred') { // Push with 'latest' tag as well
                            def latestImage = docker.build("pekker123/crud-123:latest")
                            latestImage.push()
                        }
                    }
                }
            }
        }

        stage('Deploy To EC2') {
            steps {
                script {
                    // These commands will run on the Jenkins agent itself, not directly on EC2.
                    // If your EC2 is separate, you'll need SSH or a deployment tool.
                    // Assuming this Jenkins agent *is* your EC2 instance or has Docker installed and can reach the remote Docker daemon.
                    sh 'docker rm -f $(docker ps -aq --filter ancestor=pekker123/crud-123:latest) || true' // Remove existing containers of this image
                    sh 'docker pull pekker123/crud-123:latest' // Pull the latest image
                    sh 'docker run -d -p 3000:3000 --name crud-app pekker123/crud-123:latest' // Add --name for easier management
                }
            }
        }
    }
}
