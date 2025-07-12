pipeline {
    agent any

    environment {
        // Fetch the sonar-scanner tool; it will set SCANNER_HOME automatically
        SCANNER_HOME = tool 'sonar-scanner' // Added this line back from your second Jenkinsfile block

        // Use SONAR_AUTH_TOKEN as the environment variable name
        SONAR_AUTH_TOKEN = credentials('sonar-token') 
        
        // Correct these to your actual SonarCloud organization and project key
        SONAR_ORGANIZATION = 'ashwin2701' // THIS WAS THE PROBLEM: Change back to your actual organization
        SONAR_PROJECT_KEY = 'ashwin2701_ci-jenkins' // THIS WAS THE PROBLEM: Change back to your actual project key
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
                    // Changed back to using env.SCANNER_HOME, as defined above
                    sh """${env.SCANNER_HOME}/bin/sonar-scanner \\
                        -Dsonar.organization=${env.SONAR_ORGANIZATION} \\
                        -Dsonar.projectKey=${env.SONAR_PROJECT_KEY} \\
                        -Dsonar.sources=.
                        """
                    // No need for -Dsonar.host.url=https://sonarcloud.io here as withSonarQubeEnv handles it
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                // Reinstating the timeout with a larger value, as we found the previous issue was a race condition/timeout,
                // and you confirmed the Quality Gate passes quickly on SonarCloud.
                timeout(time: 15, unit: 'MINUTES') { 
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build And Push') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-cred') { // Explicitly define Docker Hub registry URL
                        def buildNumber = env.BUILD_NUMBER ?: '1'
                        def image = docker.build("pekker123/crud-123:${buildNumber}")
                        image.push()

                        // Make sure to push the 'latest' tag as well, as your deploy step uses it
                        docker.withRegistry('https://index.docker.io/v1/', 'docker-cred') { // Re-specify registry for clarity
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
                    // Use --filter ancestor= and a specific image name for safety,
                    // or ensure you only have one app running. `docker ps -q` removes all running containers.
                    sh 'docker rm -f $(docker ps -aq --filter ancestor=pekker123/crud-123) || true'
                    sh 'docker pull pekker123/crud-123:latest'
                    sh 'docker run -d -p 3000:3000 --name crud-app pekker123/crud-123:latest'
                }
            }
        }
    }
}
