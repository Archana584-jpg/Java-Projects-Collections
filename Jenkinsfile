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
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.COMMIT_HASH = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    env.REPO_URL = sh(script: 'git config --get remote.origin.url | sed "s/.git//"', returnStdout: true).trim()                    
                    env.IS_PR = (env.CHANGE_ID != null) ? 'true' : 'false'
                    
                    echo "Commit: ${env.COMMIT_HASH}"
                    echo "Repo: ${env.REPO_URL}"
                    echo "Is PR: ${env.IS_PR}"
                    echo "CHANGE_ID: ${env.CHANGE_ID ?: 'NOT SET'}"
                    echo "CHANGE_BRANCH: ${env.CHANGE_BRANCH ?: 'NOT SET'}"
                    echo "CHANGE_TARGET: ${env.CHANGE_TARGET ?: 'NOT SET'}"
                }
            }
        }

        stage('Build Gradle Projects') {
            steps {
                script {
                    sh '''
                        echo "=== Building Gradle Projects ==="
                        
                        gradle_projects=$(find Projects -name "build.gradle" -type f 2>/dev/null)
                        
                        if [ -z "$gradle_projects" ]; then
                            echo "No Gradle projects found, skipping build"
                            exit 0
                        fi
                        
                        build_count=0
                        for gradle_file in $gradle_projects; do
                            project_dir=$(dirname "$gradle_file")
                            echo "Building: $project_dir"
                            build_count=$((build_count + 1))
                            
                            cd "$project_dir"
                            
                            if [ -f "gradlew" ]; then
                                echo "Using gradlew..."
                                ./gradlew clean build -x test --no-daemon 2>&1 | tail -30 || echo "Build failed for $project_dir, continuing..."
                            else
                                echo "gradlew not found in $project_dir, skipping"
                            fi
                            
                            cd - > /dev/null
                            
                            if [ $build_count -ge 2 ]; then
                                echo "Built 2 projects, stopping to save time"
                                break
                            fi
                        done
                        
                        echo "Build stage completed - built $build_count projects"
                    '''
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
                    find Projects -name "build.gradle" -type f 2>/dev/null | sort
                    echo ""
                    echo "=== Java Files Count ==="
                    find Projects -name "*.java" -type f 2>/dev/null | wc -l
                    echo ""
                    echo "=== Compiled Classes ==="
                    find Projects -name "*.class" -type f 2>/dev/null | wc -l
                '''
            }
        }

        stage('SonarQube Scan') {
            steps {
                script {
                    sh '''
                        export PATH=${SONAR_SCANNER_HOME}/bin:$PATH
                        
                        # Find all build directories for binaries
                        BINARIES=$(find Projects -type d \\( -name "classes" -o -name "intermediates" \\) 2>/dev/null | tr '\\n' ',' | sed 's/,$//')
                        
                        echo "Found binaries: $BINARIES"
                        echo "CHANGE_ID: '${CHANGE_ID}'"
                        echo "CHANGE_BRANCH: '${CHANGE_BRANCH}'"
                        echo "CHANGE_TARGET: '${CHANGE_TARGET}'"
                        
                        if [ ! -z "${CHANGE_ID}" ] && [ ! -z "${CHANGE_BRANCH}" ]; then
                            echo "=== Running SonarQube scan for PR ${CHANGE_ID} ==="
                            sonar-scanner \
                                -Dsonar.projectKey=${PROJECT_KEY} \
                                -Dsonar.projectName="Java Projects Collections" \
                                -Dsonar.host.url=${SONAR_HOST_URL} \
                                -Dsonar.token=${SONAR_TOKEN} \
                                -Dsonar.sources=Projects \
                                -Dsonar.sourceEncoding=UTF-8 \
                                -Dsonar.java.binaries="$BINARIES" \
                                -Dsonar.exclusions='**/build/**,**/node_modules/**,**/*.gradle,**/target/**,**/*.war,**/*.wav' \
                                -Dsonar.pullrequest.key=${CHANGE_ID} \
                                -Dsonar.pullrequest.branch=${CHANGE_BRANCH} \
                                -Dsonar.pullrequest.base=${CHANGE_TARGET}
                        else
                            echo "=== Running SonarQube scan for branch ==="
                            sonar-scanner \
                                -Dsonar.projectKey=${PROJECT_KEY} \
                                -Dsonar.projectName="Java Projects Collections" \
                                -Dsonar.host.url=${SONAR_HOST_URL} \
                                -Dsonar.token=${SONAR_TOKEN} \
                                -Dsonar.sources=Projects \
                                -Dsonar.sourceEncoding=UTF-8 \
                                -Dsonar.java.binaries="$BINARIES" \
                                -Dsonar.exclusions='**/build/**,**/node_modules/**,**/*.gradle,**/target/**,**/*.war,**/*.wav'
                        fi
                    '''
                }
            }
        }

        stage('Quality Gate Check') {
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
                                    
                                    if [ -z "$STATUS" ]; then
                                        echo "No status in response"
                                    else
                                        echo "Status: $STATUS"
                                        if [ "$STATUS" != "IN_REVIEW" ]; then
                                            echo "$STATUS"
                                            exit 0
                                        fi
                                    fi
                                    
                                    sleep 10
                                done
                                
                                echo "TIMEOUT"
                            ''',
                            returnStdout: true
                        ).trim()
                        
                        env.QG_STATUS = qgStatus
                        echo "Quality Gate Status: ${env.QG_STATUS}"
                    }
                }
            }
        }

        stage('Report to GitHub') {
            steps {
                script {
                    if (env.CHANGE_ID != null && env.CHANGE_ID != '') {
                        echo "Posting status to GitHub PR: ${env.CHANGE_ID}"
                        
                        def statusState = (env.QG_STATUS == 'OK') ? 'success' : 'failure'
                        def statusDescription = (env.QG_STATUS == 'OK') ? 'Quality gate passed ✅' : "Quality gate failed (${env.QG_STATUS})"
                        def targetUrl = "${env.SONAR_HOST_URL}/dashboard?id=${PROJECT_KEY}&pullRequest=${env.CHANGE_ID}"

                        sh '''
                            echo "=== Posting to GitHub ==="
                            echo "Commit: ${COMMIT_HASH}"
                            echo "Repo URL: ${REPO_URL}"
                            
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
                            
                            echo "GitHub Status Posted"
                        '''
                    } else {
                        echo "Not a PR build (CHANGE_ID not set), skipping GitHub status update"
                    }
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
                echo "PR ID: ${CHANGE_ID:-'NOT SET'}"
                echo "Quality Gate: ${QG_STATUS:-'NOT SET'}"
                echo "Workspace: ${WORKSPACE}"
            '''
        }

        failure {
            script {
                if (env.CHANGE_ID != null && env.CHANGE_ID != '') {
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
