pipeline {
    agent any

    options {
        timeout(time: 5, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    stages {
        stage('Detect PR') {
            steps {
                script {
                    if (env.CHANGE_ID != null && env.CHANGE_ID != '') {
                        env.BUILD_TYPE = 'PR'
                        env.PR_NUMBER = env.CHANGE_ID
                        env.TARGET_BRANCH = env.CHANGE_TARGET ?: 'main'
                        echo "✓ PR DETECTED: PR-${PR_NUMBER}"
                        echo "  Base branch: ${TARGET_BRANCH}"
                    } else {
                        env.BUILD_TYPE = 'BRANCH'
                        echo "ℹ Branch push (no PR detected)"
                    }
                }
            }
        }

        stage('Get Code Changes') {
            when {
                expression { env.BUILD_TYPE == 'PR' }
            }
            steps {
                script {
                    echo "════════════════════════════════════"
                    echo "Analyzing Changes in PR-${PR_NUMBER}"
                    echo "════════════════════════════════════"
                    
                    sh '''
                        echo "Fetching ${CHANGE_TARGET}..."
                        git fetch origin ${CHANGE_TARGET} --quiet 2>&1 || true
                    '''
                    
                    // Get all changed files - TWO DOTS (not three)
                    def allFiles = sh(
                        script: """
                            git diff --name-only origin/${env.CHANGE_TARGET}..HEAD 2>/dev/null || echo ""
                        """,
                        returnStdout: true
                    ).trim().split('\n').findAll { it.trim() }
                    
                    echo ""
                    echo "📝 ALL CHANGED FILES (${allFiles.size()}):"
                    echo "════════════════════════════════════"
                    allFiles.each { f ->
                        echo "  • $f"
                    }
                    
                    // Get code files only
                    def codeFiles = allFiles.findAll { it =~ /\.(java|js|py|html|css|xml|properties|gradle)$/ && it.trim() != '' }
                    
                    echo ""
                    echo "💻 CODE FILES (${codeFiles.size()}):"
                    echo "════════════════════════════════════"
                    if (codeFiles.size() > 0) {
                        codeFiles.each { f ->
                            echo "  ✓ $f"
                        }
                    } else {
                        echo "  (No code files changed)"
                    }
                    
                    // Show diff stats
                    echo ""
                    echo "📊 SUMMARY:"
                    echo "════════════════════════════════════"
                    sh '''
                        git diff --stat origin/${CHANGE_TARGET}..HEAD || true
                    '''
                    
                    env.CODE_FILES_COUNT = codeFiles.size().toString()
                }
            }
        }

        stage('Show Changes') {
            when {
                expression { env.BUILD_TYPE == 'PR' && env.CODE_FILES_COUNT != '0' }
            }
            steps {
                script {
                    echo ""
                    echo "📄 DETAILED CHANGES:"
                    echo "════════════════════════════════════"
                    sh '''
                        git diff origin/${CHANGE_TARGET}..HEAD --no-color
                    '''
                }
            }
        }

        stage('Summary') {
            when {
                expression { env.BUILD_TYPE == 'PR' }
            }
            steps {
                script {
                    echo ""
                    echo "════════════════════════════════════"
                    echo "SUMMARY"
                    echo "════════════════════════════════════"
                    echo "PR: #${PR_NUMBER}"
                    echo "Target: ${CHANGE_TARGET}"
                    echo "Code Files Changed: ${CODE_FILES_COUNT}"
                    echo "════════════════════════════════════"
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
