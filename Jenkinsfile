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
                    
                    def repoUrl = sh(script: 'git config --get remote.origin.url', returnStdout: true).trim()
                    repoUrl = repoUrl.replaceAll('\\.git$', '')
                    
                    def owner, repo
                    if (repoUrl.contains('github.com/')) {
                        def parts = repoUrl.split('github.com/')[1].split('/')
                        owner = parts[0]
                        repo = parts[1]
                    } else if (repoUrl.contains('github.com:')) {
                        def parts = repoUrl.split('github.com:')[1].split('/')
                        owner = parts[0]
                        repo = parts[1]
                    } else {
                        error("Invalid GitHub repo URL")
                    }
                    
                    env.GITHUB_REPO = "${owner}/${repo}"
                    env.GITHUB_API_REPO = "${GITHUB_API}/repos/${owner}/${repo}"
                    
                    echo "Repo: ${env.GITHUB_REPO}"
                    
                    // Check if this is a PR build or branch push
                    if (env.CHANGE_ID != null && env.CHANGE_ID != '') {
                        // Native PR build in Jenkins
                        env.BUILD_TYPE = 'PR'
                        env.PR_NUMBER = env.CHANGE_ID
                        env.TARGET_BRANCH = env.CHANGE_TARGET
                        env.SONAR_PR_KEY = "${SONAR_BASE_KEY}-pr-${PR_NUMBER}"
                        echo "✓ PR detected (Jenkins native): PR-${PR_NUMBER}"
                        echo "  Target: ${TARGET_BRANCH}"
                    } else {
                        // Branch push - check for open PR on GitHub
                        echo "Branch push detected: ${BRANCH_NAME}"
                        echo "Checking GitHub for open PR on this branch..."
                        
                        // FIX: Added quotes around Authorization header
                        def prResponse = sh(
                            script: '''
                                curl -s -H "Authorization: token ${GITHUB_TOKEN}" \
                                    "${GITHUB_API_REPO}/pulls?state=open&head=${GITHUB_REPO}:${BRANCH_NAME}" 2>/dev/null
                            ''',
                            returnStdout: true
                        ).trim()
                        
                        echo "GitHub API Response: ${prResponse}"
                        
                        def prNumber = null
                        def prBase = null
                        
                        try {
                            def parsed = new groovy.json.JsonSlurper().parseText(prResponse)
                            if (parsed instanceof List && parsed.size() > 0) {
                                prNumber = parsed[0].number.toString()
                                prBase = parsed[0].base.ref
                                echo "Found PR: #${prNumber} targeting ${prBase}"
                            }
                        } catch (Exception e) {
                            echo "Could not parse GitHub response: ${e.message}"
                        }
                        
                        if (prNumber && prNumber != 'null' && prNumber != '') {
                            // Found open PR
                            env.BUILD_TYPE = 'PR'
                            env.PR_NUMBER = prNumber
                            env.SONAR_PR_KEY = "${SONAR_BASE_KEY}-pr-${PR_NUMBER}"
                            env.TARGET_BRANCH = prBase ?: 'main'
                            env.CHANGE_TARGET = env.TARGET_BRANCH
                            
                            echo "✓ Found open PR: PR-${PR_NUMBER}"
                            echo "  Target: ${TARGET_BRANCH}"
                        } else {
                            // No open PR
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
                    echo "Repo: ${env.REPO_URL}"
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
                        
                        // Use TWO DOTS (not three)
                        def changedFiles = sh(
                            script: """
                                git diff --name-only origin/${targetBranch}..HEAD 2>/dev/null | sort | uniq
                            """,
                            returnStdout: true
                        ).trim().split('\n').findAll { it.trim() }
                        
                        echo "All files changed:"
                        changedFiles.each { f ->
                            echo "  - ${f}"
                        }
                        
                        // Filter for source files only
                        def sourceFiles = changedFiles.findAll { f ->
                            f =~ /\.(java|html|js|py)$/ && f.trim() != ''
                        }
                        
                        env.CHANGE_COUNT = sourceFiles.size().toString()
                        echo "✓ Source files changed: ${env.CHANGE_COUNT}"
                        
                        if (sourceFiles.size() > 0) {
                            env.CHANGED_FILES = sourceFiles.join(',')
                            echo "Files to scan:"
                            sourceFiles.each { f -> echo "  ✓ ${f}" }
                        }
                        
                        echo "════════════════════════════════════"
                        
                    } catch (Exception e) {
                        echo "⚠ Error detecting changes: ${e.message}"
                        env.CHANGE_COUNT = '0'
                    }
                }
            }
        }

        stage('Skip if No Changes') {
            when {
                expression { env.BUILD_TYPE == 'PR' && env.CHANGE_COUNT == '0' }
            }
            steps {
                sh '''
                    curl -s -X POST \
                        -H "Authorization: token ${GITHUB_TOKEN}" \
                        -H "Content-Type: application/json" \
                        -d '{"state":"success","description":"No code changes","target_url":"","context":"Jenkins/SonarQube"}' \
                        "${REPO_URL}/statuses/${COMMIT_HASH}"
                    echo "✓ GitHub status posted: No changes"
                '''
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
                    echo "ℹ To enable scanning:"
                    echo "  1. Go to GitHub"
                    echo "  2. Open PR from branch: ${BRANCH_NAME}"
                    echo "  3. Push commit again to trigger PR build"
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
                sh '''
                    EXISTS=$(curl -s -u "${SONAR_TOKEN}:" "${SONAR_HOST_URL}/api/projects/search?projects=${SONAR_PR_KEY}" | grep -c '"key"' || echo 0)
                    if [ "$EXISTS" -eq 0 ]; then
                        echo "Creating SonarQube project: ${SONAR_PR_KEY}"
                        curl -s -X POST -u "${SONAR_TOKEN}:" \
                            -d "project=${SONAR_PR_KEY}&name=${SONAR_PR_KEY}" \
                            "${SONAR_HOST_URL}/api/projects/create"
                        echo "✓ Project created"
                    else
                        echo "✓ Project already exists"
                    fi
                '''
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
                            -Dsonar.exclusions='**/build/**,**/target/**,**/node_modules/**'
                        
                        echo "✓ Scan submitted to SonarQube"
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
                        echo "Polling quality gate status..."
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
                        echo "Quality Gate Result: ${env.QG_STATUS}"
                    }
                }
            }
        }

        stage('Delete Project on Pass') {
            when {
                expression { env.BUILD_TYPE == 'PR' && env.CHANGE_COUNT != '0' && env.QG_STATUS == 'OK' }
            }
            steps {
                sh '''
                    echo "✓✓✓ Quality gate PASSED"
                    echo "Deleting temp project: ${SONAR_PR_KEY}"
                    curl -s -X POST -u "${SONAR_TOKEN}:" \
                        -d "project=${SONAR_PR_KEY}" \
                        "${SONAR_HOST_URL}/api/projects/delete"
                    echo "✓ Project deleted"
                '''
            }
        }

        stage('Preserve Project on Fail') {
            when {
                expression { env.BUILD_TYPE == 'PR' && env.CHANGE_COUNT != '0' && env.QG_STATUS != 'OK' }
            }
            steps {
                sh '''
                    echo "✗✗✗ Quality gate FAILED"
                    echo "Preserving project for review: ${SONAR_PR_KEY}"
                    echo "Review at: ${SONAR_HOST_URL}/dashboard?id=${SONAR_PR_KEY}"
                '''
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
                        desc = 'Quality gate analysis timeout'
                        targetUrl = "${SONAR_HOST_URL}/dashboard?id=${SONAR_PR_KEY}"
                    } else {
                        state = 'failure'
                        desc = 'Quality gate failed - review issues'
                        targetUrl = "${SONAR_HOST_URL}/dashboard?id=${SONAR_PR_KEY}"
                    }
                    
                    sh '''
                        curl -s -X POST \
                            -H "Authorization: token ${GITHUB_TOKEN}" \
                            -H "Content-Type: application/json" \
                            -d '{"state":"''' + state + '''","description":"''' + desc + '''","target_url":"''' + targetUrl + '''","context":"Jenkins/SonarQube"}' \
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
                            -d '{"state":"error","description":"Pipeline execution failed","target_url":"${BUILD_URL}","context":"Jenkins/Pipeline"}' \
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
