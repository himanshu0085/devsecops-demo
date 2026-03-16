pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Run Gitleaks') {
            steps {
                sh '''
                gitleaks detect --source . \
                --report-format sarif \
                --report-path gitleaks.sarif
                '''
            }
        }

        stage('Run Trivy') {
            steps {
                sh '''
                trivy fs . \
                --format sarif \
                --output trivy.sarif
                '''
            }
        }

        stage('Show Reports') {
            steps {
                sh 'ls -l'
            }
        }

    }
}
