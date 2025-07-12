pipeline {
    agent any

    environment {
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
                    script {
                        def scannerHome = tool 'sonar-scanner'
                        sh """
                            ${scannerHome}/bin/sonar-scanner \\
                              -Dsonar.organization=${env.SONAR_ORGANIZATION} \\
                              -Dsonar.projectKey=${env.SONAR_PROJECT_KEY} \\
                              -Dsonar.sources=.
                        """
                    }
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                script {
                    echo 'Skipping waitForQualityGate due to SonarCloud webhook limitations (free plan)'
                    echo 'Waiting manually for SonarCloud analysis to complete (adjust duration as needed)...'
                    sleep time: 30, unit: 'SECONDS'
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

                        docker.withRegistry('', 'docker-cred') {
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
                    sh 'docker rm -f $(docker ps -aq --filter ancestor=pekker123/crud-123:latest) || true'
