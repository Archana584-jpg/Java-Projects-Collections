pipeline {
    agent any

    environment {
        GITHUB_TOKEN = credentials('archana-sonar')
        SONAR_TOKEN = credentials('sonar-token')
        SONAR_HOST_URL = 'http://15.206.213.78:9000'
        SONAR_BASE_KEY = 'kamar-taj'
        SONAR_PR_KEY = "${SONAR_BASE_KEY}-pr-${CHANGE_ID}"
        SONAR_SCANNER_HOME = "${WORKSPACE}/sonar-scanner"
    }

    options {
        timeout(time: 15, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '3'))
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.COMMIT_HASH = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    
                    // Get repo URL from git config and convert to API format
                    def repoUrl = sh(script: 'git config --get remote.origin.url', returnStdout: true).trim()
                    
                    // Convert https://github.com/owner/repo.git to https://api.github.com/repos/owner/repo
                    if (repoUrl.contains('github.com')) {
                        repoUrl = repoUrl
                            .replaceAll('https://', '')
                            .replaceAll('git@github.com:', '')
                            .replaceAll('.git$', '')
                            .replaceAll('github.com/', '')
                        env.REPO_URL = "https://api.github.com/repos/${repoUrl}"
                    } else {
                        env.REPO_URL = repoUrl.replaceAll('.git$', '')
                    }
                    
                    echo "PR: ${CHANGE_ID} | Commit: ${env.COMMIT_HASH}"
                    echo "Repo API URL: ${env.REPO_URL}"
                }
            }
        }

        stage('Detect Changed Files') {
            when {
                expression { env.CHANGE_ID != null && env.CHANGE_ID != '' }
            }
            steps {
                script {
                    def changeCount = sh(
                        script: '''
                            # Fetch the target branch safely
                            git fetch origin ${CHANGE_TARGET} 2>/dev/null || true
                            
                            # Count changed source files
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
                }
            }
        }

        stage('Skip if No Changes') {
            when {
                expression { env.CHANGE_ID != null && env.CHANGE_ID != '' && env.CHANGE_COUNT == '0' }
            }
            steps {
                script {
                    echo "No code changes detected — skipping scan"
                    sh '''
                        curl -s -X POST \
                            -H "Authorization: token ${GITHUB_TOKEN}" \
                            -H "Content-Type: application/json" \
                            -d '{"state":"success","description":"No code changes","target_url":"","context":"Jenkins/SonarQube"}' \
                            "${REPO_URL}/statuses/${COMMIT_HASH}"
                        echo "GitHub status posted"
                    '''
                }
            }
        }

        stage('Download Scanner') {
            when {
                expression { env.CHANGE_COUNT != '0' }
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
                        echo "Scanner ready"
                    else
                        echo "Scanner already cached"
                    fi
                '''
            }
        }

        stage('Create SonarQube Project') {
            when {
                expression { env.CHANGE_COUNT != '0' }
            }
            steps {
                script {
                    echo "Checking if temp project ${SONAR_PR_KEY} exists..."
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
                                "${SONAR_HOST_URL}/api/projects/create" | head -1
                            echo "Project created"
                        '''
                    } else {
                        echo "Project ${SONAR_PR_KEY} already exists"
                    }
                }
            }
        }

        stage('SonarQube Scan') {
            when {
                expression { env.CHANGE_COUNT != '0' }
            }
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    sh '''
                        export PATH=${SONAR_SCANNER_HOME}/bin:$PATH
                        
                        SOURCES="${CHANGED_FILES}"
                        if [ -z "$SOURCES" ]; then
                            SOURCES="Projects"
                        fi
                        
                        echo "Scanning: $SOURCES to project ${SONAR_PR_KEY}"
                        
                        sonar-scanner \
                            -Dsonar.projectKey="${SONAR_PR_KEY}" \
                            -Dsonar.projectName="${SONAR_PR_KEY}" \
                            -Dsonar.host.url="${SONAR_HOST_URL}" \
                            -Dsonar.token="${SONAR_TOKEN}" \
                            -Dsonar.sources="$SOURCES" \
                            -Dsonar.sourceEncoding=UTF-8 \
                            -Dsonar.exclusions='**/build/**,**/node_modules/**,**/*.gradle,**/target/**,**/intermediates/**'
                        
                        echo "Scan complete"
                    '''
                }
            }
        }

        stage('Check Quality Gate') {
            when {
                expression { env.CHANGE_COUNT != '0' }
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
                expression { env.CHANGE_COUNT != '0' && env.QG_STATUS == 'OK' }
            }
            steps {
                script {
                    echo "✓ Quality gate PASSED — deleting ${SONAR_PR_KEY}"
                    sh '''
                        curl -s -X POST \
                            -u "${SONAR_TOKEN}:" \
                            -d "project=${SONAR_PR_KEY}" \
                            "${SONAR_HOST_URL}/api/projects/delete"
                        echo "Project deleted"
                    '''
                    env.PROJECT_DELETED = 'true'
                }
            }
        }

        stage('Preserve Project on Fail') {
            when {
                expression { env.CHANGE_COUNT != '0' && env.QG_STATUS != 'OK' }
            }
            steps {
                script {
                    echo "✗ Quality gate FAILED — keeping ${SONAR_PR_KEY} for review"
                    env.PROJECT_DELETED = 'false'
                }
            }
        }

        stage('Report to GitHub') {
            when {
                expression { env.CHANGE_ID != null && env.CHANGE_ID != '' }
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
                    
                    echo "Posting to GitHub: ${state} - ${desc}"
                    
                    sh '''
                        PAYLOAD=$(cat <<EOF
{
  "state":"''' + state + '''",
  "description":"''' + desc + '''",
  "target_url":"''' + targetUrl + '''",
  "context":"Jenkins/SonarQube"
}
EOF
                        )
                        
                        echo "URL: ${REPO_URL}/statuses/${COMMIT_HASH}"
                        echo "Payload: $PAYLOAD"
                        
                        curl -s -X POST \
                            -H "Authorization: token ${GITHUB_TOKEN}" \
                            -H "Content-Type: application/json" \
                            -d "$PAYLOAD" \
                            "${REPO_URL}/statuses/${COMMIT_HASH}"
                        
                        echo "GitHub status updated"
                    '''
                }
            }
        }
    }

    post {
        failure {
            script {
                if (env.CHANGE_ID != null && env.CHANGE_ID != '') {
                    echo "Pipeline failed — posting error to GitHub"
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
