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

        stage('Quality Gate Check (Sleep workaround)') {
