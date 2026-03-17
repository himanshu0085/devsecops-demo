pipeline {
    agent any

    environment {
        GITHUB_TOKEN = credentials('github-token')
        REPO = "himanshu0085/devsecops-demo"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Run Gitleaks Scan') {
            steps {
                sh '''
                gitleaks detect --source . \
                --report-format sarif \
                --report-path gitleaks.sarif || true
                '''
            }
        }

        stage('Run Trivy Scan') {
            steps {
                sh '''
                trivy fs . --scanners vuln --format table > trivy-report.txt

                trivy fs . \
                --format template \
                --template @/var/lib/jenkins/trivy-templates/html.tpl \
                --output trivy-report.html
                '''
            }
        }

        stage('Publish Trivy Report to Jenkins UI') {
    steps {
        publishHTML([
            reportDir: '.',
            reportFiles: 'trivy-report.html',
            reportName: 'Trivy Security Report',
            keepAll: true,
            alwaysLinkToLastBuild: true,
            allowMissing: false
        ])
    }
}
        stage('Create Security Summary') {
            steps {
                sh '''
                echo "DevSecOps Scan Summary" > security-summary.txt
                echo "" >> security-summary.txt
                echo "Gitleaks: ✔ Scan Completed" >> security-summary.txt
                echo "Trivy: ✔ Scan Completed" >> security-summary.txt
                '''
            }
        }

        stage('Post PR Comment') {
            steps {
                sh '''
                export GH_TOKEN=$GITHUB_TOKEN

                gh pr comment 1 \
                --repo $REPO \
                --body "$(cat security-summary.txt)" || true
                '''
            }
        }

        stage('Update Commit Status') {
            steps {
                sh '''
                curl -X POST \
                -H "Authorization: token $GITHUB_TOKEN" \
                -H "Accept: application/vnd.github.v3+json" \
                https://api.github.com/repos/$REPO/statuses/$GIT_COMMIT \
                -d '{
                  "state": "success",
                  "context": "security/trivy",
                  "description": "Security scans completed"
                }'
                '''
            }
        }

    }
}
