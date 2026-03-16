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
        trivy fs . \
        --format sarif \
        --output trivy.sarif

        trivy fs . \
        --format template \
        --template "@$HOME/trivy-templates/html.tpl" \
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

        stage('Upload Security Results to GitHub') {
            steps {
                sh '''
                gh auth login --with-token <<< $GITHUB_TOKEN

                gh api \
                  --method POST \
                  -H "Accept: application/vnd.github+json" \
                  /repos/$REPO/code-scanning/sarifs \
                  -f sarif=@trivy.sarif
                '''
            }
        }

        stage('Update GitHub Commit Status') {
            steps {
                sh '''
                curl -X POST \
                -H "Authorization: token $GITHUB_TOKEN" \
                -H "Accept: application/vnd.github.v3+json" \
                https://api.github.com/repos/$REPO/statuses/$GIT_COMMIT \
                -d '{
                  "state": "success",
                  "context": "security/trivy",
                  "description": "Trivy security scan completed"
                }'
                '''
            }
        }

    }
}
