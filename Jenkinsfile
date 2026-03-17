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

                gitleaks detect --source . \
                --report-format json \
                --report-path gitleaks.json || true
                '''
            }
        }

        stage('Run Trivy Scan') {
            steps {
                sh '''
                # Table (for summary)
                trivy fs . --format table > trivy-report.txt

                # SARIF (for GitHub Security tab)
                trivy fs . --format sarif --output trivy.sarif

                # HTML (for Jenkins UI)
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
                if grep -q '"StartLine"' gitleaks.json; then
                    STATUS="❌ Secrets detected"
                else
                    STATUS="✅ No secrets found"
                fi

                echo "<html><body><h2>Gitleaks Report</h2>" > gitleaks-report.html
                echo "<p>$STATUS</p><pre>" >> gitleaks-report.html
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
                echo "🔐 DevSecOps Scan Summary" > security-summary.txt
                echo "" >> security-summary.txt

                # Gitleaks
                if grep -q '"StartLine"' gitleaks.json; then
                    echo "❌ Gitleaks: Secrets detected" >> security-summary.txt
                else
                    echo "✅ Gitleaks: No secrets found" >> security-summary.txt
                fi

                # Trivy
                if grep -q "CRITICAL\\|HIGH" trivy-report.txt; then
                    echo "❌ Trivy: Vulnerabilities found" >> security-summary.txt
                else
                    echo "✅ Trivy: No vulnerabilities found" >> security-summary.txt
                fi
                '''
            }
        }

        stage('Upload SARIF to GitHub Security') {
            steps {
                sh '''
                export GH_TOKEN=$GITHUB_TOKEN

                echo "Uploading Trivy SARIF..."
                gh api \
                  --method POST \
                  -H "Accept: application/vnd.github+json" \
                  /repos/$REPO/code-scanning/sarifs \
                  -f sarif=@trivy.sarif || true

                echo "Uploading Gitleaks SARIF..."
                gh api \
                  --method POST \
                  -H "Accept: application/vnd.github+json" \
                  /repos/$REPO/code-scanning/sarifs \
                  -f sarif=@gitleaks.sarif || true
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
                    echo "No PR found, skipping PR comment"
                fi
                '''
            }
        }

        stage('Update Commit Status') {
            steps {
                sh '''
                if grep -q '"StartLine"' gitleaks.json; then
                    STATUS="failure"
                    DESC="❌ Secrets detected by Gitleaks"
                elif grep -q "CRITICAL\\|HIGH" trivy-report.txt; then
                    STATUS="failure"
                    DESC="❌ Vulnerabilities found by Trivy"
                else
                    STATUS="success"
                    DESC="✅ No issues found"
                fi

                curl -X POST \
                -H "Authorization: token $GITHUB_TOKEN" \
                -H "Accept: application/vnd.github.v3+json" \
                https://api.github.com/repos/$REPO/statuses/$GIT_COMMIT \
                -d "{
                  \\"state\\": \\"$STATUS\\",
                  \\"context\\": \\"security/devsecops\\",
                  \\"description\\": \\"$DESC\\"
                }"
                '''
            }
        }

    }
}
