pipeline {
    agent any

    environment {
        GITHUB_TOKEN = credentials('archana-sonar')
        SONAR_TOKEN = credentials('sonar-token')
        SONAR_HOST_URL = 'http://15.206.213.78:9000'
        SONAR_BASE_KEY = 'kamar-taj'
        SONAR_SCANNER_HOME = "${WORKSPACE}/sonar-scanner"
        GITHUB_API = 'https://api.github.com'
    }

    options {
        timeout(time: 15, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '3'))
        disableConcurrentBuilds()
    }

    stages {
        stage('Detect Build Type and PR') {
            steps {
                script {
                    echo "════════════════════════════════════"
                    echo "Detecting Build Type"
                    echo "════════════════════════════════════"
                    
                    // Parse repo owner/name from URL
                    def repoUrl = sh(script: 'git config --get remote.origin.url', returnStdout: true).trim()
                    repoUrl = repoUrl.replaceAll('\\.git$', '')
                    def repoMatch = (repoUrl =~ /github\.com[:/]([^/]+)\/(.+)$/)
                    
                    if (!repoMatch) {
                        echo "ERROR: Could not parse repo URL: ${repoUrl}"
                        error("Invalid GitHub repo URL")
                    }
                    
                    def owner = repoMatch[0][1]
                    def repo = repoMatch[0][2]
                    env.GITHUB_REPO = "${owner}/${repo}"
                    env.GITHUB_API_REPO = "${GITHUB_API}/repos/${owner}/${repo}"
                    
                    echo "Repo: ${env.GITHUB_REPO}"
                    
                    // Check if this is a PR build or branch push
                    if (env.CHANGE_ID != null && env.CHANGE_ID != '') {
                        // Already a PR build in Jenkins
                        env.BUILD_TYPE = 'PR'
                        env.PR_NUMBER = env.CHANGE_ID
                        env.TARGET_BRANCH = env.CHANGE_TARGET
                        env.SONAR_PR_KEY = "${SONAR_BASE_KEY}-pr-${PR_NUMBER}"
                        echo "✓ Pull Request detected (Jenkins native): PR-${PR_NUMBER}"
                        echo "  Target: ${TARGET_BRANCH}"
                    } else {
                        // Branch push - check if there's an open PR for this branch
                        echo "Branch push detected: ${BRANCH_NAME}"
                        echo "Checking GitHub for open PR on this branch..."
                        
                        def prResponse = sh(
                            script: '''
                                curl -s -H "Authorization: token ${GITHUB_TOKEN}" \
                                    "${GITHUB_API_REPO}/pulls?state=open&head=${GITHUB_REPO}:${BRANCH_NAME}" | \
                                    jq -r '.[] | select(.state=="open") | .number' | head -1
                            ''',
                            returnStdout: true
                        ).trim()
                        
                        if (prResponse && prResponse != 'null' && prResponse != '') {
                            // Found open PR for this branch
                            env.BUILD_TYPE = 'PR'
                            env.PR_NUMBER = prResponse
                            env.SONAR_PR_KEY = "${SONAR_BASE_KEY}-pr-${PR_NUMBER}"
                            
                            // Get PR details to find target branch
                            def prDetails = sh(
                                script: '''
                                    curl -s -H "Authorization: token ${GITHUB_TOKEN}" \
                                        "${GITHUB_API_REPO}/pulls/${PR_NUMBER}" | \
                                        jq -r '.base.ref'
                                ''',
                                returnStdout: true
                            ).trim()
                            
                            env.TARGET_BRANCH = prDetails ?: 'main'
                            env.CHANGE_TARGET = env.TARGET_BRANCH
                            
                            echo "✓ Found open PR: PR-${PR_NUMBER}"
                            echo "  Target: ${TARGET_BRANCH}"
                        } else {
                            // No open PR for this branch
                            env.BUILD_TYPE = 'BRANCH'
                            env.SONAR_PR_KEY = "${SONAR_BASE_KEY}-${BRANCH_NAME}".replaceAll('/', '-')
                            echo "ℹ Branch push with no open PR"
                            echo "  To trigger PR scanning, open a PR for this branch"
                        }
                    }
                    
                    echo "════════════════════════════════════"
                    echo "Build Type: ${env.BUILD_TYPE}"
                    echo "PR Number: ${env.PR_NUMBER ?: 'N/A'}"
                    echo "SonarQube Key: ${env.SONAR_PR_KEY}"
                    echo "════════════════════════════════════"
                }
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.COMMIT_HASH = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    
                    def repoUrl = sh(script: 'git config --get remote.origin.url', returnStdout: true).trim()
                    repoUrl = repoUrl.replaceAll('\\.git$', '')
                    
                    if (repoUrl.contains('github.com')) {
                        repoUrl = repoUrl.replaceAll('https://github.com/', '')
                                        .replaceAll('git@github.com:', '')
                        env.REPO_URL = "https://api.github.com/repos/${repoUrl}"
                    } else {
                        env.REPO_URL = repoUrl
                    }
                    
                    echo "Commit: ${env.COMMIT_HASH}"
                    echo "Repo URL: ${env.REPO_URL}"
                    echo "Branch: ${BRANCH_NAME}"
                }
            }
        }

        stage('Detect Changed Files') {
            when {
                expression { env.BUILD_TYPE == 'PR' }
            }
            steps {
                script {
                    try {
                        echo "════════════════════════════════════"
                        echo "Detecting Changed Files"
                        echo "════════════════════════════════════"
                        
                        def targetBranch = env.TARGET_BRANCH ?: 'main'
                        echo "Target branch: ${targetBranch}"
                        echo "Current branch: ${BRANCH_NAME}"
                        
                        // Fetch target branch
                        sh '''
                            echo "Fetching origin/${TARGET_BRANCH}..."
                            git fetch origin ${TARGET_BRANCH} --quiet 2>&1 || true
                        '''
                        
                        // Get changed files
                        def changedFiles = []
                        
                        try {
                            changedFiles = sh(
                                script: """
                                    git diff --name-only origin/${targetBranch}...HEAD 2>/dev/null | sort | uniq
                                """,
                                returnStdout: true
                            ).trim().split('\n').findAll { it.trim() }
                            echo "Method 1 (remote comparison): ${changedFiles.size()} files"
                        } catch (Exception e1) {
                            echo "Method 1 failed, trying Method 2..."
                            try {
                                changedFiles = sh(
                                    script: '''
                                        git diff --name-only HEAD~1...HEAD 2>/dev/null | sort | uniq
                                    ''',
                                    returnStdout: true
                                ).trim().split('\n').findAll { it.trim() }
                                echo "Method 2 (HEAD~1): ${changedFiles.size()} files"
                            } catch (Exception e2) {
                                echo "Method 2 failed, using Method 3..."
                                changedFiles = sh(
                                    script: '''
                                        git show --name-only --pretty="" HEAD | sort | uniq
                                    ''',
                                    returnStdout: true
                                ).trim().split('\n').findAll { it.trim() }
                                echo "Method 3 (HEAD show): ${changedFiles.size()} files"
                            }
                        }
                        
                        echo "All changed files:"
                        changedFiles.each { echo "  - $it" }
                        
                        // Filter for source files
                        def sourceFiles = changedFiles.findAll { it =~ /\.(java|html|js|py)$/ }
                        
                        env.CHANGE_COUNT = sourceFiles.size().toString()
                        echo "✓ Source files changed: ${env.CHANGE_COUNT}"
                        
                        if (sourceFiles.size() > 0) {
                            env.CHANGED_FILES = sourceFiles.join(',')
                            echo "Files to scan:"
                            sourceFiles.each { echo "  ✓ $it" }
                        }
                        
                        echo "════════════════════════════════════"
                        
                    } catch (Exception e) {
                        echo "⚠ Error detecting changes: ${e.message}"
                        env.CHANGE_COUNT = '0'
                    }
                }
            }
        }

        stage('Skip if No PR Changes') {
            when {
                expression { env.BUILD_TYPE == 'PR' && env.CHANGE_COUNT == '0' }
            }
            steps {
                script {
                    echo "✓ No code changes detected — skipping scan"
                    sh '''
                        curl -s -X POST \
                            -H "Authorization: token ${GITHUB_TOKEN}" \
                            -H "Content-Type: application/json" \
                            -d '{"state":"success","description":"No code changes","target_url":"","context":"Jenkins/SonarQube"}' \
                            "${REPO_URL}/statuses/${COMMIT_HASH}"
                    '''
                }
            }
        }

        stage('Branch Without PR') {
            when {
                expression { env.BUILD_TYPE == 'BRANCH' }
            }
            steps {
                script {
                    echo "ℹ This is a branch push with no open PR"
                    echo "ℹ Skipping PR-specific scanning"
                    echo "ℹ To enable scanning, open a PR for branch: ${BRANCH_NAME}"
                }
            }
        }

        stage('Download SonarQube Scanner') {
            when {
                expression { env.BUILD_TYPE == 'PR' && env.CHANGE_COUNT != '0' }
            }
            steps {
                sh '''
                    if [ ! -f "${SONAR_SCANNER_HOME}/bin/sonar-scanner" ]; then
                        mkdir -p ${WORKSPACE}
                        cd ${WORKSPACE}
                        echo "Downloading SonarQube Scanner..."
                        wget -q https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip
                        unzip -q sonar-scanner-cli-5.0.1.3006-linux.zip
                        mv sonar-scanner-5.0.1.3006-linux ${SONAR_SCANNER_HOME}
                        rm sonar-scanner-cli-5.0.1.3006-linux.zip
                        echo "✓ Scanner ready"
                    else
                        echo "✓ Scanner already cached"
                    fi
                '''
            }
        }

        stage('Create SonarQube Temp Project') {
            when {
                expression { env.BUILD_TYPE == 'PR' && env.CHANGE_COUNT != '0' }
            }
            steps {
                script {
                    echo "Checking if project ${SONAR_PR_KEY} exists..."
                    def projectExists = sh(
                        script: '''
                            curl -s -u "${SONAR_TOKEN}:" "${SONAR_HOST_URL}/api/projects/search?projects=${SONAR_PR_KEY}" | grep -q '"key":"${SONAR_PR_KEY}"'
                            echo $?
                        ''',
                        returnStdout: true
                    ).trim()
                    
                    if (projectExists == '1') {
                        echo "Creating temp SonarQube project: ${SONAR_PR_KEY}"
                        sh '''
                            curl -s -X POST \
                                -u "${SONAR_TOKEN}:" \
                                -d "project=${SONAR_PR_KEY}&name=${SONAR_PR_KEY}&visibility=private" \
                                "${SONAR_HOST_URL}/api/projects/create"
                            echo "✓ Project created"
                        '''
                    } else {
                        echo "✓ Project already exists"
                    fi
                }
            }
        }

        stage('SonarQube Scan') {
            when {
                expression { env.BUILD_TYPE == 'PR' && env.CHANGE_COUNT != '0' }
            }
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    sh '''
                        export PATH=${SONAR_SCANNER_HOME}/bin:$PATH
                        
                        SOURCES="${CHANGED_FILES}"
                        if [ -z "$SOURCES" ]; then
                            SOURCES="Projects"
                        fi
                        
                        echo "════════════════════════════════════"
                        echo "SonarQube Scan"
                        echo "════════════════════════════════════"
                        echo "Project: ${SONAR_PR_KEY}"
                        echo "Sources: $SOURCES"
                        echo "════════════════════════════════════"
                        
                        sonar-scanner \
                            -Dsonar.projectKey="${SONAR_PR_KEY}" \
                            -Dsonar.projectName="${SONAR_PR_KEY}" \
                            -Dsonar.host.url="${SONAR_HOST_URL}" \
                            -Dsonar.token="${SONAR_TOKEN}" \
                            -Dsonar.sources="$SOURCES" \
                            -Dsonar.sourceEncoding=UTF-8 \
                            -Dsonar.exclusions='**/build/**,**/node_modules/**,**/*.gradle,**/target/**,**/intermediates/**'
                        
                        echo "✓ Scan submitted"
                    '''
                }
            }
        }

        stage('Wait for Quality Gate') {
            when {
                expression { env.BUILD_TYPE == 'PR' && env.CHANGE_COUNT != '0' }
            }
            steps {
                timeout(time: 3, unit: 'MINUTES') {
                    script {
                        echo "Polling quality gate..."
                        def qgStatus = sh(
                            script: '''
                                for i in {1..18}; do
                                    STATUS=$(curl -s -u "${SONAR_TOKEN}:" "${SONAR_HOST_URL}/api/qualitygates/project_status?projectKey=${SONAR_PR_KEY}" | grep -o '"status":"[A-Z]*' | cut -d'"' -f4)
                                    if [ "$STATUS" = "OK" ] || [ "$STATUS" = "ERROR" ]; then
                                        echo "$STATUS"
                                        exit 0
                                    fi
                                    echo "Attempt $i/18..."
                                    sleep 10
                                done
                                echo "TIMEOUT"
                            ''',
                            returnStdout: true
                        ).trim()
                        env.QG_STATUS = qgStatus
                        echo "Quality Gate: ${env.QG_STATUS}"
                    }
                }
            }
        }

        stage('Delete Project on Pass') {
            when {
                expression { env.BUILD_TYPE == 'PR' && env.CHANGE_COUNT != '0' && env.QG_STATUS == 'OK' }
            }
            steps {
                script {
                    echo "✓✓✓ Quality gate PASSED"
                    echo "Deleting project: ${SONAR_PR_KEY}"
                    sh '''
                        curl -s -X POST \
                            -u "${SONAR_TOKEN}:" \
                            -d "project=${SONAR_PR_KEY}" \
                            "${SONAR_HOST_URL}/api/projects/delete"
                        echo "✓ Deleted"
                    '''
                    env.PROJECT_DELETED = 'true'
                }
            }
        }

        stage('Preserve Project on Fail') {
            when {
                expression { env.BUILD_TYPE == 'PR' && env.CHANGE_COUNT != '0' && env.QG_STATUS != 'OK' }
            }
            steps {
                script {
                    echo "✗✗✗ Quality gate FAILED"
                    echo "Keeping project: ${SONAR_PR_KEY}"
                    echo "Review at: ${SONAR_HOST_URL}/dashboard?id=${SONAR_PR_KEY}"
                    env.PROJECT_DELETED = 'false'
                }
            }
        }

        stage('Report to GitHub') {
            when {
                expression { env.BUILD_TYPE == 'PR' }
            }
            steps {
                script {
                    def state, desc, targetUrl
                    
                    if (env.CHANGE_COUNT == '0') {
                        state = 'success'
                        desc = 'No code changes'
                        targetUrl = ''
                    } else if (env.QG_STATUS == 'OK') {
                        state = 'success'
                        desc = 'Quality gate passed ✓'
                        targetUrl = "${SONAR_HOST_URL}/dashboard?id=${SONAR_BASE_KEY}"
                    } else if (env.QG_STATUS == 'TIMEOUT') {
                        state = 'failure'
                        desc = 'Quality gate timeout'
                        targetUrl = "${SONAR_HOST_URL}/dashboard?id=${SONAR_PR_KEY}"
                    } else {
                        state = 'failure'
                        desc = 'Quality gate failed'
                        targetUrl = "${SONAR_HOST_URL}/dashboard?id=${SONAR_PR_KEY}"
                    }
                    
                    sh '''
                        PAYLOAD=$(cat <<'PAYLOAD_EOF'
{
  "state":"''' + state + '''",
  "description":"''' + desc + '''",
  "target_url":"''' + targetUrl + '''",
  "context":"Jenkins/SonarQube"
}
PAYLOAD_EOF
                        )
                        
                        curl -s -X POST \
                            -H "Authorization: token ${GITHUB_TOKEN}" \
                            -H "Content-Type: application/json" \
                            -d "$PAYLOAD" \
                            "${REPO_URL}/statuses/${COMMIT_HASH}"
                        echo "✓ GitHub status updated"
                    '''
                }
            }
        }
    }

    post {
        failure {
            script {
                if (env.BUILD_TYPE == 'PR') {
                    sh '''
                        curl -s -X POST \
                            -H "Authorization: token ${GITHUB_TOKEN}" \
                            -H "Content-Type: application/json" \
                            -d '{"state":"error","description":"Pipeline failed","target_url":"${BUILD_URL}","context":"Jenkins/Pipeline"}' \
                            "${REPO_URL}/statuses/${COMMIT_HASH}"
                    '''
                }
            }
        }
        
        always {
            cleanWs()
        }
    }
}
