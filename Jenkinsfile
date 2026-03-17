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
                --report-format json \
                --report-path gitleaks.json || true
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

        stage('Convert Gitleaks Report to HTML') {
            steps {
                sh '''
                echo "<html><body><h2>Gitleaks Report</h2><pre>" > gitleaks-report.html
                cat gitleaks.json >> gitleaks-report.html
                echo "</pre></body></html>" >> gitleaks-report.html
                '''
            }
        }

        stage('Publish Reports to Jenkins UI') {
            steps {
                publishHTML([
                    reportDir: '.',
                    reportFiles: 'trivy-report.html',
                    reportName: 'Trivy Security Report',
                    keepAll: true,
                    alwaysLinkToLastBuild: true,
                    allowMissing: false
                ])

                publishHTML([
                    reportDir: '.',
                    reportFiles: 'gitleaks-report.html',
                    reportName: 'Gitleaks Report',
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

                # Gitleaks check
                if grep -q '"StartLine"' gitleaks.json; then
                    echo "Gitleaks: ⚠ Secrets detected" >> security-summary.txt
                else
                    echo "Gitleaks: ✔ No secrets found" >> security-summary.txt
                fi

                # Trivy check
                if grep -q "CRITICAL\\|HIGH" trivy-report.txt; then
                    echo "Trivy: ⚠ Vulnerabilities found" >> security-summary.txt
                else
                    echo "Trivy: ✔ No vulnerabilities found" >> security-summary.txt
                fi
                '''
            }
        }

        stage('Post PR Comment') {
            steps {
                sh '''
                export GH_TOKEN=$GITHUB_TOKEN

                PR_NUMBER=$(gh pr list --repo $REPO --limit 1 --json number -q '.[0].number')

                if [ ! -z "$PR_NUMBER" ]; then
                    gh pr comment $PR_NUMBER \
                    --repo $REPO \
                    --body "$(cat security-summary.txt)"
                else
                    echo "No PR found, skipping comment"
                fi
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
                  "context": "security/devsecops",
                  "description": "Gitleaks & Trivy scans completed"
                }'
                '''
            }
        }

    }
}
