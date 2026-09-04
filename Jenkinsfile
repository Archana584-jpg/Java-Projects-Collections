pipeline {
    agent any

    options {
        timeout(time: 10, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        skipDefaultCheckout(false)
    }

    stages {

        // ============================================================
        // 1. DETECT WHETHER THIS IS A PR BUILD
        // ============================================================
        stage('Detect PR') {
            steps {
                script {

                    echo "=============================================="
                    echo "JENKINS BUILD INFORMATION"
                    echo "=============================================="

                    echo "CHANGE_ID      = ${env.CHANGE_ID ?: 'NOT SET'}"
                    echo "CHANGE_TARGET  = ${env.CHANGE_TARGET ?: 'NOT SET'}"
                    echo "CHANGE_BRANCH  = ${env.CHANGE_BRANCH ?: 'NOT SET'}"
                    echo "BRANCH_NAME    = ${env.BRANCH_NAME ?: 'NOT SET'}"
                    echo "GIT_COMMIT     = ${env.GIT_COMMIT ?: 'NOT SET'}"

                    echo "=============================================="

                    if (env.CHANGE_ID?.trim()) {

                        env.BUILD_TYPE = 'PR'
                        env.PR_NUMBER = env.CHANGE_ID
                        env.TARGET_BRANCH = env.CHANGE_TARGET ?: 'main'
                        env.SOURCE_BRANCH = env.CHANGE_BRANCH ?: env.BRANCH_NAME

                        echo ""
                        echo "PR DETECTED"
                        echo "----------------------------------------------"
                        echo "PR Number      : ${env.PR_NUMBER}"
                        echo "Source Branch  : ${env.SOURCE_BRANCH}"
                        echo "Target Branch  : ${env.TARGET_BRANCH}"
                        echo "Jenkins Job    : ${env.BRANCH_NAME}"
                        echo "=============================================="

                    } else {

                        env.BUILD_TYPE = 'BRANCH'

                        echo ""
                        echo "NORMAL BRANCH BUILD"
                        echo "----------------------------------------------"
                        echo "Branch         : ${env.BRANCH_NAME}"
                        echo "=============================================="
                    }
                }
            }
        }


        // ============================================================
        // 2. CHECKOUT PR CODE
        // ============================================================
        stage('Checkout PR') {

            when {
                expression {
                    env.BUILD_TYPE == 'PR'
                }
            }

            steps {

                checkout scm

                sh '''
                    set -e

                    echo "=============================================="
                    echo "CHECKOUT INFORMATION"
                    echo "=============================================="

                    echo "Current commit:"
                    git rev-parse HEAD

                    echo ""
                    echo "Current branch:"
                    git branch --show-current

                    echo ""
                    echo "Fetching target branch:"
                    echo "${CHANGE_TARGET}"

                    git fetch origin \
                        ${CHANGE_TARGET}:refs/remotes/origin/${CHANGE_TARGET}

                    echo ""
                    echo "Available branches:"
                    git branch -a

                    echo "=============================================="
                '''
            }
        }


        // ============================================================
        // 3. DETECT CHANGED FILES
        // ============================================================
        stage('Detect Changed Files') {

            when {
                expression {
                    env.BUILD_TYPE == 'PR'
                }
            }

            steps {

                script {

                    echo ""
                    echo "=============================================="
                    echo "DETECTING PR CHANGES"
                    echo "=============================================="

                    def changedFilesOutput = sh(
                        script: """
                            git diff --name-only \
                            origin/${env.TARGET_BRANCH}...HEAD
                        """,
                        returnStdout: true
                    ).trim()

                    def allFiles = []

                    if (changedFilesOutput) {
                        allFiles = changedFilesOutput
                            .split('\\n')
                            .findAll { it.trim() }
                    }

                    echo ""
                    echo "ALL CHANGED FILES: ${allFiles.size()}"
                    echo "----------------------------------------------"

                    if (allFiles.size() > 0) {

                        allFiles.each { file ->
                            echo "  ${file}"
                        }

                    } else {

                        echo "  No files changed"
                    }


                    // ------------------------------------------------
                    // CODE FILES
                    // ------------------------------------------------

                    def codeExtensions = [
                        'java',
                        'js',
                        'jsx',
                        'ts',
                        'tsx',
                        'py',
                        'go',
                        'php',
                        'rb',
                        'cs',
                        'cpp',
                        'c',
                        'h',
                        'html',
                        'css',
                        'scss',
                        'xml',
                        'properties',
                        'gradle',
                        'groovy',
                        'yml',
                        'yaml',
                        'json'
                    ]

                    def codeFiles = allFiles.findAll { file ->

                        def lowerFile = file.toLowerCase()

                        codeExtensions.any { extension ->
                            lowerFile.endsWith(".${extension}")
                        }
                    }


                    echo ""
                    echo "CODE FILES: ${codeFiles.size()}"
                    echo "----------------------------------------------"

                    if (codeFiles.size() > 0) {

                        codeFiles.each { file ->
                            echo "  ✓ ${file}"
                        }

                    } else {

                        echo "  No code files changed"
                    }


                    // ------------------------------------------------
                    // STORE VARIABLES FOR NEXT STAGES
                    // ------------------------------------------------

                    env.ALL_FILES_COUNT = allFiles.size().toString()
                    env.CODE_FILES_COUNT = codeFiles.size().toString()

                    // Store changed files safely
                    env.CHANGED_CODE_FILES = codeFiles.join(',')

                    echo ""
                    echo "=============================================="
                    echo "CHANGE SUMMARY"
                    echo "=============================================="
                    echo "PR Number       : ${env.PR_NUMBER}"
                    echo "Source Branch   : ${env.SOURCE_BRANCH}"
                    echo "Target Branch   : ${env.TARGET_BRANCH}"
                    echo "Total Files     : ${env.ALL_FILES_COUNT}"
                    echo "Code Files      : ${env.CODE_FILES_COUNT}"
                    echo "=============================================="
                }
            }
        }


        // ============================================================
        // 4. DIFF STAT
        // ============================================================
        stage('Show Diff Summary') {

            when {
                expression {
                    env.BUILD_TYPE == 'PR'
                }
            }

            steps {

                sh """
                    echo "=============================================="
                    echo "PR DIFF SUMMARY"
                    echo "=============================================="

                    git diff --stat \
                        origin/${TARGET_BRANCH}...HEAD

                    echo "=============================================="
                """
            }
        }


        // ============================================================
        // 5. SHOW ACTUAL CHANGES
        // ============================================================
        stage('Show Changes') {

            when {
                expression {
                    env.BUILD_TYPE == 'PR' &&
                    env.CODE_FILES_COUNT != '0'
                }
            }

            steps {

                sh """
                    echo "=============================================="
                    echo "CODE CHANGES"
                    echo "=============================================="

                    git diff --no-color \
                        origin/${TARGET_BRANCH}...HEAD

                    echo "=============================================="
                """
            }
        }


        // ============================================================
        // 6. PR SUMMARY
        // ============================================================
        stage('PR Summary') {

            when {
                expression {
                    env.BUILD_TYPE == 'PR'
                }
            }

            steps {

                echo ""
                echo "=============================================="
                echo "FINAL PR SUMMARY"
                echo "=============================================="
                echo ""
                echo "PR Number       : ${env.PR_NUMBER}"
                echo "Source Branch   : ${env.SOURCE_BRANCH}"
                echo "Target Branch   : ${env.TARGET_BRANCH}"
                echo "Total Files     : ${env.ALL_FILES_COUNT}"
                echo "Code Files      : ${env.CODE_FILES_COUNT}"
                echo ""
                echo "Changed Code:"
                echo "${env.CHANGED_CODE_FILES ?: 'None'}"
                echo ""
                echo "=============================================="
            }
        }
    }


    // ================================================================
    // POST
    // ================================================================
    post {

        success {
            echo "✓ PR pipeline completed successfully"
        }

        failure {
            echo "✗ PR pipeline failed"
        }

        always {
            echo "Cleaning Jenkins workspace..."
            cleanWs()
        }
    }
}
