pipeline {
    agent any

    environment {
        GITHUB_TOKEN = credentials('archana-sonar')
        SONAR_TOKEN = credentials('sonar-token')
        SONAR_HOST_URL = 'http://15.206.213.78:9000'
        SONAR_BASE_KEY = 'kamar-taj'
        SONAR_SCANNER_HOME = "${WORKSPACE}/sonar-scanner"
    }

    options {
        timeout(time: 15, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '3'))
        disableConcurrentBuilds()
    }

    stages {
        stage('Detect Build Type') {
            steps {
                script {
                    if (env.CHANGE_ID != null && env.CHANGE_ID != '') {
                        env.BUILD_TYPE = 'PR'
                        env.SONAR_PR_KEY = "${SONAR_BASE_KEY}-pr-${CHANGE_ID}"
                        echo "✓ Pull Request detected: PR-${CHANGE_ID}"
                    } else {
                        env.BUILD_TYPE = 'BRANCH'
                        env.SONAR_PR_KEY = "${SONAR_BASE_KEY}-${BRANCH_NAME}".replaceAll('/', '-')
                        echo "ℹ Branch push detected: ${BRANCH_NAME}"
                    }
                    echo "Build Type: ${env.BUILD_TYPE}"
                    echo "SonarQube Key: ${env.SONAR_PR_KEY}"
                }
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.COMMIT_HASH = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    
                    // Get repo URL from git config
                    def repoUrl = sh(script: 'git config --get remote.origin.url', returnStdout: true).trim()
                    
                    // Remove .git suffix using Groovy (safer than sed)
                    repoUrl = repoUrl.replaceAll('\\.git$', '')
                    
                    // Convert to GitHub API format
                    if (repoUrl.contains('github.com')) {
                        // Handle both https:// and git@ formats
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
                        def changeCount = sh(
                            script: '''
                                git fetch origin ${CHANGE_TARGET} 2>/dev/null || true
                                git diff --name-only origin/${CHANGE_TARGET}...HEAD 2>/dev/null | grep -E '\\.(java|html|js|py)$' | wc -l
                            ''',
                            returnStdout: true
                        ).trim().toInteger()
                        
                        env.CHANGE_COUNT = changeCount.toString()
                        echo "Changed source files: ${env.CHANGE_COUNT}"
                        
                        if (changeCount > 0) {
                            env.CHANGED_FILES = sh(
                                script: '''
                                    git diff --name-only origin/${CHANGE_TARGET}...HEAD 2>/dev/null | grep -E '\\.(java|html|js|py)$' | tr '\\n' ','
                                ''',
                                returnStdout: true
                            ).trim().replaceAll(',$', '')
                            echo "Files to scan: ${env.CHANGED_FILES}"
                        }
                    } catch (Exception e) {
                        echo "Warning: Could not detect changes - ${e.message}"
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
                        echo "GitHub status posted: No changes"
                    '''
                }
            }
        }

        stage('Branch Push Detected') {
            when {
                expression { env.BUILD_TYPE == 'BRANCH' }
            }
            steps {
                script {
                    echo "ℹ This is a branch push, not a PR"
                    echo "ℹ Skipping PR-specific scanning and gating"
                    echo "ℹ To test, create a PR from this branch"
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
                            RESPONSE=$(curl -s -X POST \
                                -u "${SONAR_TOKEN}:" \
                                -d "project=${SONAR_PR_KEY}&name=${SONAR_PR_KEY}&visibility=private" \
                                "${SONAR_HOST_URL}/api/projects/create")
                            echo "✓ Project created: ${SONAR_PR_KEY}"
                        '''
                    } else {
                        echo "✓ Project ${SONAR_PR_KEY} already exists - will reuse"
                    }
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
                        echo "SonarQube Scan Details"
                        echo "════════════════════════════════════"
                        echo "Project Key: ${SONAR_PR_KEY}"
                        echo "Sources: $SOURCES"
                        echo "Server: ${SONAR_HOST_URL}"
                        echo "════════════════════════════════════"
                        
                        sonar-scanner \
                            -Dsonar.projectKey="${SONAR_PR_KEY}" \
                            -Dsonar.projectName="${SONAR_PR_KEY}" \
                            -Dsonar.host.url="${SONAR_HOST_URL}" \
                            -Dsonar.token="${SONAR_TOKEN}" \
                            -Dsonar.sources="$SOURCES" \
                            -Dsonar.sourceEncoding=UTF-8 \
                            -Dsonar.exclusions='**/build/**,**/node_modules/**,**/*.gradle,**/target/**,**/intermediates/**'
                        
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
                                    PERCENT=$((i * 5))
                                    echo "Attempt $i/18 ($PERCENT% complete)..."
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

        stage('Delete Temp Project on Pass') {
            when {
                expression { env.BUILD_TYPE == 'PR' && env.CHANGE_COUNT != '0' && env.QG_STATUS == 'OK' }
            }
            steps {
                script {
                    echo "✓✓✓ Quality gate PASSED"
                    echo "Deleting temp project: ${SONAR_PR_KEY}"
                    sh '''
                        curl -s -X POST \
                            -u "${SONAR_TOKEN}:" \
                            -d "project=${SONAR_PR_KEY}" \
                            "${SONAR_HOST_URL}/api/projects/delete"
                        echo "✓ Project deleted"
                    '''
                    env.PROJECT_DELETED = 'true'
                }
            }
        }

        stage('Preserve Temp Project on Fail') {
            when {
                expression { env.BUILD_TYPE == 'PR' && env.CHANGE_COUNT != '0' && env.QG_STATUS != 'OK' }
            }
            steps {
                script {
                    echo "✗✗✗ Quality gate FAILED"
                    echo "Preserving project for manual review: ${SONAR_PR_KEY}"
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
                        desc = 'No code changes detected'
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
                    
                    echo "Posting to GitHub:"
                    echo "  State: ${state}"
                    echo "  Description: ${desc}"
                    echo "  URL: ${env.REPO_URL}/statuses/${COMMIT_HASH}"
                    
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
                    echo "✗ Pipeline failed — posting error to GitHub"
                    sh '''
                        curl -s -X POST \
                            -H "Authorization: token ${GITHUB_TOKEN}" \
                            -H "Content-Type: application/json" \
                            -d '{"state":"error","description":"Pipeline execution failed","target_url":"${BUILD_URL}","context":"Jenkins/Pipeline"}' \
                            "${REPO_URL}/statuses/${COMMIT_HASH}"
                        echo "✓ Error status posted to GitHub"
                    '''
                }
            }
        }
        
        always {
            cleanWs()
        }
    }
}
