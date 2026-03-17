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
                trivy fs . --format table > trivy-report.txt

                trivy fs . --format sarif --output trivy.sarif

                trivy fs . \
                --format template \
                --template @/var/lib/jenkins/trivy-templates/html.tpl \
                --output trivy-report.html
                '''
            }
        }

        stage('Convert Reports to HTML') {
            steps {
                sh '''
                if grep -q '"StartLine"' gitleaks.json; then
                    STATUS="❌ Secrets detected"
                else
                    STATUS="✅ No secrets found"
                fi

                echo "<html><body><h2>Gitleaks Report</h2><p>$STATUS</p><pre>" > gitleaks-report.html
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

        stage('Create PR Comment') {
            steps {
                sh '''
                echo "## 🔐 DevSecOps Scan Report" > comment.md
                echo "" >> comment.md

                echo "### 🛑 Gitleaks Findings" >> comment.md
                grep '"Description"' gitleaks.json >> comment.md || echo "No secrets found" >> comment.md

                echo "" >> comment.md
                echo "### 🚨 Trivy Findings" >> comment.md
                grep "CRITICAL\\|HIGH" trivy-report.txt >> comment.md || echo "No vulnerabilities found" >> comment.md
                '''
            }
        }

        stage('Upload SARIF to GitHub') {
            steps {
                sh '''
                export GH_TOKEN=$GITHUB_TOKEN

                # Detect branch safely
                BRANCH=$(git remote show origin | grep 'HEAD branch' | awk '{print $NF}')

                echo "Detected branch: $BRANCH"
                echo "Commit: $GIT_COMMIT"

                # Encode Trivy SARIF
                gzip -c trivy.sarif > trivy.sarif.gz
                TRIVY_BASE64=$(base64 -w 0 trivy.sarif.gz)

                # Encode Gitleaks SARIF
                gzip -c gitleaks.sarif > gitleaks.sarif.gz
                GITLEAKS_BASE64=$(base64 -w 0 gitleaks.sarif.gz)

                echo "Uploading Trivy SARIF..."
                gh api \
                  --method POST \
                  -H "Accept: application/vnd.github+json" \
                  /repos/$REPO/code-scanning/sarifs \
                  -f commit_sha=$GIT_COMMIT \
                  -f ref=refs/heads/$BRANCH \
                  -f sarif="$TRIVY_BASE64"

                echo "Uploading Gitleaks SARIF..."
                gh api \
                  --method POST \
                  -H "Accept: application/vnd.github+json" \
                  /repos/$REPO/code-scanning/sarifs \
                  -f commit_sha=$GIT_COMMIT \
                  -f ref=refs/heads/$BRANCH \
                  -f sarif="$GITLEAKS_BASE64"
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
                    --body-file comment.md
                else
                    echo "No PR found"
                fi
                '''
            }
        }

        stage('Update Commit Status') {
            steps {
                sh '''
                if grep -q '"StartLine"' gitleaks.json; then
                    STATUS="failure"
                    DESC="Secrets detected"
                elif grep -q "CRITICAL\\|HIGH" trivy-report.txt; then
                    STATUS="failure"
                    DESC="Vulnerabilities found"
                else
                    STATUS="success"
                    DESC="No issues found"
                fi

                curl -s -X POST \
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
