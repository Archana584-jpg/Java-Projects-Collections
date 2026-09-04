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
                        echo "Fetching ${TARGET_BRANCH}..."
                        git fetch origin ${TARGET_BRANCH} --quiet 2>&1 || true
                    '''
                    
                    // Get all changed files
                    def allFiles = sh(
                        script: """
                            git diff --name-only origin/${env.TARGET_BRANCH}..HEAD 2>/dev/null || echo ""
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
                    def codeFiles = allFiles.findAll { it =~ /\.(java|js|py|html|css|xml|properties|gradle|maven)$/ && it.trim() != '' }
                    
                    echo ""
                    echo "💻 CODE FILES CHANGED (${codeFiles.size()}):"
                    echo "════════════════════════════════════"
                    if (codeFiles.size() > 0) {
                        codeFiles.each { f ->
                            echo "  ✓ $f"
                        }
                    } else {
                        echo "  (No code files changed - only config/docs)"
                    }
                    
                    // Show actual diff
                    echo ""
                    echo "📊 DETAILED CHANGES:"
                    echo "════════════════════════════════════"
                    sh '''
                        git diff --stat origin/${TARGET_BRANCH}..HEAD || true
                    '''
                    
                    echo ""
                    echo "📄 FILE-BY-FILE CHANGES:"
                    echo "════════════════════════════════════"
                    codeFiles.each { f ->
                        echo ""
                        echo "FILE: $f"
                        echo "───────────────────────────────────"
                        sh """
                            git diff origin/${env.TARGET_BRANCH}..HEAD -- "$f" | head -50 || true
                        """
                    }
                    
                    env.CODE_FILES_COUNT = codeFiles.size().toString()
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
                    echo "Target: ${TARGET_BRANCH}"
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
