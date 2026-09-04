pipeline {
    agent any

    environment {
        GITHUB_TOKEN = credentials('archana-sonar')
        SONAR_TOKEN = credentials('sonar-token')
        SONAR_HOST_URL = 'http://15.206.213.78:9000'
        PROJECT_KEY = 'java-projects-collections'
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
                    env.REPO_URL = sh(script: 'git config --get remote.origin.url | sed "s/.git//"', returnStdout: true).trim()
                }
            }
        }

        stage('Check PR Changes') {
            when {
                expression { env.CHANGE_ID != null && env.CHANGE_ID != '' }
            }
            steps {
                script {
                    def changeCount = sh(
                        script: '''
                            git fetch origin ${CHANGE_TARGET} 2>/dev/null || true
                            git diff --name-only origin/${CHANGE_TARGET}...HEAD | grep -E '\\.(java|html|js|py)$' | wc -l
                        ''',
                        returnStdout: true
                    ).trim().toInteger()
                    
                    env.CHANGE_COUNT = changeCount.toString()
                    echo "Changed files: ${env.CHANGE_COUNT}"
                }
            }
        }

        stage('Download Scanner') {
            steps {
                sh '''
                    if [ ! -f "${SONAR_SCANNER_HOME}/bin/sonar-scanner" ]; then
                        mkdir -p ${WORKSPACE}
                        cd ${WORKSPACE}
                        wget -q https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip
                        unzip -q sonar-scanner-cli-5.0.1.3006-linux.zip
                        mv sonar-scanner-5.0.1.3006-linux ${SONAR_SCANNER_HOME}
                        rm sonar-scanner-cli-5.0.1.3006-linux.zip
                    fi
                '''
            }
        }

        stage('SonarQube Scan') {
            when {
                expression {
                    if (env.CHANGE_ID != null && env.CHANGE_ID != '') {
                        return env.CHANGE_COUNT.toInteger() > 0
                    }
                    return true
                }
            }
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    sh '''
                        export PATH=${SONAR_SCANNER_HOME}/bin:$PATH
                        
                        SOURCES="Projects"
                        
                        if [ ! -z "${CHANGE_ID}" ]; then
                            SOURCES=$(git diff --name-only origin/${CHANGE_TARGET}...HEAD | grep -E '\\.(java|html|js|py)$' | tr '\\n' ',')
                            SOURCES=$(echo "$SOURCES" | sed 's/,$//')
                        fi
                        
                        sonar-scanner \
                            -Dsonar.projectKey=${PROJECT_KEY} \
                            -Dsonar.projectName="Java Projects Collections" \
                            -Dsonar.host.url=${SONAR_HOST_URL} \
                            -Dsonar.token=${SONAR_TOKEN} \
                            -Dsonar.sources="$SOURCES" \
                            -Dsonar.sourceEncoding=UTF-8 \
                            -Dsonar.exclusions='**/build/**,**/node_modules/**,**/*.gradle,**/target/**'
                    '''
                }
            }
        }

        stage('Quality Gate') {
            when {
                expression {
                    if (env.CHANGE_ID != null && env.CHANGE_ID != '') {
                        return env.CHANGE_COUNT.toInteger() > 0
                    }
                    return true
                }
            }
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    script {
                        def qgStatus = sh(
                            script: '''
                                for i in {1..12}; do
                                    STATUS=$(curl -s -u ${SONAR_TOKEN}: "${SONAR_HOST_URL}/api/qualitygates/project_status?projectKey=${PROJECT_KEY}" | grep -o '"status":"[A-Z]*' | cut -d'"' -f4)
                                    if [ "$STATUS" = "OK" ] || [ "$STATUS" = "ERROR" ]; then
                                        echo "$STATUS"
                                        exit 0
                                    fi
                                    sleep 10
                                done
                                echo "TIMEOUT"
                            ''',
                            returnStdout: true
                        ).trim()
                        env.QG_STATUS = qgStatus
                    }
                }
            }
        }

        stage('GitHub Status') {
            when {
                expression { env.CHANGE_ID != null && env.CHANGE_ID != '' }
            }
            steps {
                script {
                    def state
                    def desc
                    
                    if (env.CHANGE_COUNT.toInteger() == 0) {
                        state = "success"
                        desc = "No code changes"
                    } else if (env.QG_STATUS == "OK") {
                        state = "success"
                        desc = "Quality gate passed"
                    } else {
                        state = "failure"
                        desc = "Quality gate failed"
                    }
                    
                    sh '''
                        curl -X POST \
                            -H "Authorization: token ${GITHUB_TOKEN}" \
                            -H "Content-Type: application/json" \
                            -d '{"state":"''' + state + '''","description":"''' + desc + '''","target_url":"${SONAR_HOST_URL}/dashboard?id=${PROJECT_KEY}","context":"Jenkins/SonarQube"}' \
                            ${REPO_URL}/statuses/${COMMIT_HASH}
                    '''
                }
            }
        }
    }

    post {
        failure {
            script {
                if (env.CHANGE_ID != null && env.CHANGE_ID != '') {
                    sh '''
                        curl -X POST \
                            -H "Authorization: token ${GITHUB_TOKEN}" \
                            -H "Content-Type: application/json" \
                            -d '{"state":"error","description":"Pipeline failed","target_url":"${BUILD_URL}","context":"Jenkins/Pipeline"}' \
                            ${REPO_URL}/statuses/${COMMIT_HASH}
                    '''
                }
            }
        }
    }
}
