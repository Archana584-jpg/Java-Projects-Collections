pipeline {
    agent any


    environment {
        GITHUB_TOKEN = credentials('archana-sonar')
        SONAR_TOKEN = credentials('sonar-token')
        SONAR_HOST_URL = 'http://15.206.213.78:9000'
        PROJECT_KEY = 'java-projects-collections'
        GIT_BRANCH = "${env.CHANGE_BRANCH ?: env.GIT_BRANCH}"
        SONAR_SCANNER_HOME = "${WORKSPACE}/sonar-scanner"
    }

    options {
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.COMMIT_HASH = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    env.REPO_URL = sh(script: "git config --get remote.origin.url | sed 's/.git//'", returnStdout: true).trim()
                    echo "Commit: ${env.COMMIT_HASH}"
                    echo "Repo: ${env.REPO_URL}"
                }
            }
        }

        stage('Download SonarQube Scanner') {
            steps {
                script {
                    sh '''
                        if [ ! -d "${SONAR_SCANNER_HOME}" ]; then
                            mkdir -p ${WORKSPACE}
                            cd ${WORKSPACE}
                            wget -q https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip
                            unzip -q sonar-scanner-cli-5.0.1.3006-linux.zip
                            mv sonar-scanner-5.0.1.3006-linux ${SONAR_SCANNER_HOME}
                            rm sonar-scanner-cli-5.0.1.3006-linux.zip
                        fi
                        ${SONAR_SCANNER_HOME}/bin/sonar-scanner --version
                    '''
                }
            }
        }

        stage('List Projects') {
            steps {
                sh '''
                    echo "=== Found Gradle Projects ==="
                    find Projects -name "build.gradle" -type f | sort
                    echo ""
                    echo "=== Java Files Count ==="
                    find Projects -name "*.java" -type f | wc -l
                '''
            }
        }

        stage('SonarQube Scan') {
            when {
                expression { env.CHANGE_ID != null }
            }
            steps {
                script {
                    sh '''
                        export PATH=${SONAR_SCANNER_HOME}/bin:$PATH
                        
                        sonar-scanner \
                            -Dsonar.projectKey=${PROJECT_KEY} \
                            -Dsonar.projectName="Java Projects Collections" \
                            -Dsonar.host.url=${SONAR_HOST_URL} \
                            -Dsonar.login=${SONAR_TOKEN} \
                            -Dsonar.sources=Projects \
                            -Dsonar.sourceEncoding=UTF-8 \
                            -Dsonar.exclusions='**/build/**,**/node_modules/**,**/*.gradle' \
                            -Dsonar.pullrequest.key=${CHANGE_ID} \
                            -Dsonar.pullrequest.branch=${CHANGE_BRANCH} \
                            -Dsonar.pullrequest.base=${CHANGE_TARGET}
                    '''
                }
            }
        }

        stage('Quality Gate Check') {
            when {
                expression { env.CHANGE_ID != null }
            }
            steps {
                script {
                    timeout(time: 3, unit: 'MINUTES') {
                        def qgStatus = sh(
                            script: '''
                                for i in {1..18}; do
                                    echo "Checking Quality Gate (attempt $i/18)..."
                                    
                                    RESPONSE=$(curl -s -u ${SONAR_TOKEN}: \
                                        "${SONAR_HOST_URL}/api/qualitygates/project_status?projectKey=${PROJECT_KEY}")
                                    
                                    STATUS=$(echo "$RESPONSE" | grep -o '"status":"[^"]*' | cut -d'"' -f4)
                                    
                                    echo "Response: $RESPONSE"
                                    echo "Status: $STATUS"
                                    
                                    if [ "$STATUS" != "IN_REVIEW" ] && [ ! -z "$STATUS" ]; then
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
                        echo "Quality Gate Status: ${qgStatus}"
                    }
                }
            }
        }

        stage('Report to GitHub') {
            when {
                expression { env.CHANGE_ID != null }
            }
            steps {
                script {
                    def statusState = (env.QG_STATUS == 'OK') ? 'success' : 'failure'
                    def statusDescription = (env.QG_STATUS == 'OK') ? 'Quality gate passed ✅' : "Quality gate failed (${env.QG_STATUS})"
                    def targetUrl = "${env.SONAR_HOST_URL}/dashboard?id=${PROJECT_KEY}&pullRequest=${CHANGE_ID}"

                    sh '''
                        curl -X POST \
                            -H "Authorization: token ${GITHUB_TOKEN}" \
                            -H "Content-Type: application/json" \
                            -H "Accept: application/vnd.github.v3+json" \
                            -d '{
                                "state": "''' + statusState + '''",
                                "description": "''' + statusDescription + '''",
                                "target_url": "''' + targetUrl + '''",
                                "context": "Jenkins/SonarQube"
                            }' \
                            ${REPO_URL}/statuses/${COMMIT_HASH}
                        
                        echo "GitHub Status Posted: ${statusState}"
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
                echo "=== Build Summary ==="
                echo "Commit: ${COMMIT_HASH}"
                echo "Branch: ${GIT_BRANCH}"
                echo "Quality Gate: ${QG_STATUS}"
                echo "Workspace: ${WORKSPACE}"
            '''
        }

        failure {
            script {
                if (env.CHANGE_ID != null) {
                    sh '''
                        curl -X POST \
                            -H "Authorization: token ${GITHUB_TOKEN}" \
                            -H "Content-Type: application/json" \
                            -H "Accept: application/vnd.github.v3+json" \
                            -d '{
                                "state": "error",
                                "description": "Pipeline failed",
                                "target_url": "${BUILD_URL}",
                                "context": "Jenkins/Pipeline"
                            }' \
                            ${REPO_URL}/statuses/${COMMIT_HASH}
                    '''
                }
            }
        }
    }
}
