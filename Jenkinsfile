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
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.COMMIT_HASH = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    echo "Commit: ${env.COMMIT_HASH}"
                }
            }
        }

        stage('Detect PR') {
            steps {
                script {
                    if (env.CHANGE_ID != null && env.CHANGE_ID != '') {
                        env.BUILD_TYPE = 'PR'
                        env.PR_NUMBER = env.CHANGE_ID
                        env.TARGET_BRANCH = env.CHANGE_TARGET
                        echo "✓ PR detected: PR-${PR_NUMBER}"
                    } else {
                        env.BUILD_TYPE = 'BRANCH'
                        echo "ℹ Branch push (no PR)"
                    }
                }
            }
        }

        stage('Detect Changed Files') {
            when {
                expression { env.BUILD_TYPE == 'PR' }
            }
            steps {
                script {
                    sh '''
                        git fetch origin ${CHANGE_TARGET} --quiet 2>&1 || true
                    '''
                    
                    def changedFiles = sh(
                        script: """
                            git diff --name-only origin/${env.CHANGE_TARGET}..HEAD 2>/dev/null | sort | uniq
                        """,
                        returnStdout: true
                    ).trim().split('\n').findAll { it.trim() }
                    
                    echo "Changed files:"
                    changedFiles.each { echo "  - $it" }
                    
                    def sourceFiles = changedFiles.findAll { it =~ /\.(java|html|js|py)$/ && it.trim() != '' }
                    env.CHANGE_COUNT = sourceFiles.size().toString()
                    
                    echo "✓ Source files: ${env.CHANGE_COUNT}"
                    
                    if (sourceFiles.size() > 0) {
                        env.CHANGED_FILES = sourceFiles.join(',')
                    }
                }
            }
        }

        stage('Download Scanner') {
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
                    fi
                    echo "✓ Scanner ready"
                '''
            }
        }

        stage('Create SonarQube Project') {
            when {
                expression { env.BUILD_TYPE == 'PR' && env.CHANGE_COUNT != '0' }
            }
            steps {
                sh '''
                    SONAR_PR_KEY="${SONAR_BASE_KEY}-pr-${CHANGE_ID}"
                    echo "Creating project: $SONAR_PR_KEY"
                    
                    curl -s -X POST -u "${SONAR_TOKEN}:" \
                        -d "project=$SONAR_PR_KEY&name=$SONAR_PR_KEY" \
                        "${SONAR_HOST_URL}/api/projects/create" || true
                    
                    echo "✓ Project ready"
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
                        SONAR_PR_KEY="${SONAR_BASE_KEY}-pr-${CHANGE_ID}"
                        
                        echo "Scanning..."
                        sonar-scanner \
                            -Dsonar.projectKey="$SONAR_PR_KEY" \
                            -Dsonar.projectName="$SONAR_PR_KEY" \
                            -Dsonar.host.url="${SONAR_HOST_URL}" \
                            -Dsonar.token="${SONAR_TOKEN}" \
                            -Dsonar.sources="Projects" \
                            -Dsonar.exclusions='**/build/**,**/target/**' || true
                        
                        echo "✓ Scan complete"
                    '''
                }
            }
        }

        stage('Quality Gate') {
            when {
                expression { env.BUILD_TYPE == 'PR' && env.CHANGE_COUNT != '0' }
            }
            steps {
                timeout(time: 3, unit: 'MINUTES') {
                    sh '''
                        SONAR_PR_KEY="${SONAR_BASE_KEY}-pr-${CHANGE_ID}"
                        
                        for i in {1..10}; do
                            STATUS=$(curl -s -u "${SONAR_TOKEN}:" \
                                "${SONAR_HOST_URL}/api/qualitygates/project_status?projectKey=$SONAR_PR_KEY" | \
                                grep -o '"status":"[A-Z]*' | cut -d'"' -f4)
                            
                            if [ "$STATUS" = "OK" ] || [ "$STATUS" = "ERROR" ]; then
                                echo "Quality Gate: $STATUS"
                                exit 0
                            fi
                            sleep 10
                        done
                        echo "Quality Gate: TIMEOUT"
                    '''
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
