pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        SONAR_AUTH_TOKEN = credentials('sonar-token')
        SONAR_ORGANIZATION = 'ashwin2701' // Keep these correct!
        SONAR_PROJECT_KEY = 'ashwin2701_ci-jenkins' // Keep these correct!
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
                    sh """${env.SCANNER_HOME}/bin/sonar-scanner \\
                        -Dsonar.organization=${env.SONAR_ORGANIZATION} \\
                        -Dsonar.projectKey=${env.SONAR_PROJECT_KEY} \\
                        -Dsonar.sources=.
                        """
                }
            }
        }

        // REINSTATE THIS STAGE - This is how your pipeline waits for the Quality Gate
        stage('Quality Gate Check') {
            steps {
                timeout(time: 15, unit: 'MINUTES') { // Set a generous timeout (e.g., 15 minutes)
                    waitForQualityGate abortPipeline: true
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

                        docker.withRegistry('https://index.docker.io/v1/', 'docker-cred') {
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
                    sh 'docker rm -f $(docker ps -aq --filter ancestor=pekker123/crud-123) || true'
                    sh 'docker pull pekker123/crud-123:latest'
                    sh 'docker run -d -p 3000:3000 --name crud-app pekker123/crud-123:latest'
                }
            }
        }
    }
}
