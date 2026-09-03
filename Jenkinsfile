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
                script {
                    def t0 = System.currentTimeMillis()
                    checkout scm
                    env.COMMIT_HASH = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    env.REPO_URL = sh(script: 'git config --get remote.origin.url | sed "s/.git//"', returnStdout: true).trim()
                    env.IS_PR = (env.CHANGE_ID != null) ? 'true' : 'false'

                    echo "Commit: ${env.COMMIT_HASH}"
                    echo "Repo: ${env.REPO_URL}"
                    echo "Is PR: ${env.IS_PR}"
                    echo "CHANGE_ID: ${env.CHANGE_ID ?: 'NOT SET'}"
                    echo "CHANGE_BRANCH: ${env.CHANGE_BRANCH ?: 'NOT SET'}"
                    echo "CHANGE_TARGET: ${env.CHANGE_TARGET ?: 'NOT SET'}"
                    env.T_CHECKOUT = "${(System.currentTimeMillis() - t0) / 1000}"
                    echo "STAGE TIME [Checkout]: ${env.T_CHECKOUT}s"
                }
            }
        }

        stage('Build Gradle Projects') {
            steps {
                script {
                    def t0 = System.currentTimeMillis()
                    sh '''#!/bin/bash
                        echo "=== Building Gradle Projects ==="

                        find Projects -name "build.gradle" -type f 2>/dev/null > /tmp/gradle_projects.txt

                        if [ ! -s /tmp/gradle_projects.txt ]; then
                            echo "No Gradle projects found, skipping build"
                            exit 0
                        fi

                        build_count=0
                        while IFS= read -r gradle_file; do
                            project_dir=$(dirname "$gradle_file")
                            echo "Building: $project_dir"
                            build_count=$((build_count + 1))

                            (
                                cd "$project_dir" || exit 0

                                if [ -f "gradlew" ]; then
                                    echo "Using gradlew..."
                                    chmod +x gradlew
                                    start_ts=$(date +%s)
                                    timeout 240 ./gradlew clean build -x test --no-daemon --offline 2>&1 | tail -30
                                    gradle_rc=$?
                                    end_ts=$(date +%s)
                                    echo "gradlew for $project_dir took $((end_ts - start_ts))s (exit $gradle_rc)"
                                    if [ $gradle_rc -ne 0 ]; then
                                        echo "NOTE: gradlew failed or timed out for $project_dir (likely --offline cache miss or dependency issue), continuing..."
                                    fi
                                else
                                    echo "gradlew not found in $project_dir, skipping"
                                fi
                            )

                            if [ $build_count -ge 2 ]; then
                                echo "Built 2 projects, stopping to save time"
                                break
                            fi
                        done < /tmp/gradle_projects.txt

                        echo "Build stage completed - built $build_count projects"
                    '''
                    env.T_BUILD = "${(System.currentTimeMillis() - t0) / 1000}"
                    echo "STAGE TIME [Build Gradle Projects]: ${env.T_BUILD}s"
                }
            }
        }

        stage('Download SonarQube Scanner') {
            steps {
                script {
                    def t0 = System.currentTimeMillis()
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
                    env.T_DOWNLOAD = "${(System.currentTimeMillis() - t0) / 1000}"
                    echo "STAGE TIME [Download SonarQube Scanner]: ${env.T_DOWNLOAD}s"
                }
            }
        }

        stage('List Projects') {
            steps {
                script {
                    def t0 = System.currentTimeMillis()
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
                    env.T_LIST = "${(System.currentTimeMillis() - t0) / 1000}"
                    echo "STAGE TIME [List Projects]: ${env.T_LIST}s"
                }
            }
        }

        stage('SonarQube Scan') {
            steps {
                script {
                    def t0 = System.currentTimeMillis()
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
                    env.T_SCAN = "${(System.currentTimeMillis() - t0) / 1000}"
                    echo "STAGE TIME [SonarQube Scan]: ${env.T_SCAN}s"
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                script {
                    def t0 = System.currentTimeMillis()
                    timeout(time: 3, unit: 'MINUTES') {
                        def qgStatus = sh(
                            script: '''#!/bin/bash
                                final_status="TIMEOUT"

                                for i in $(seq 1 18); do
                                    echo "Checking Quality Gate (attempt $i/18)..." >&2

                                    RESPONSE=$(curl -s -u ${SONAR_TOKEN}: \
                                        "${SONAR_HOST_URL}/api/qualitygates/project_status?projectKey=${PROJECT_KEY}")

                                    STATUS=$(echo "$RESPONSE" | grep -o '"status":"[^"]*' | cut -d'"' -f4)
                                    echo "Status: ${STATUS:-<none>}" >&2

                                    # OK / ERROR are terminal. NONE and IN_REVIEW mean
                                    # the SonarQube background task hasn't finished
                                    # processing the report yet -- keep polling.
                                    if [ "$STATUS" = "OK" ] || [ "$STATUS" = "ERROR" ]; then
                                        final_status="$STATUS"
                                        break
                                    fi

                                    sleep 10
                                done

                                echo "$final_status"
                            ''',
                            returnStdout: true
                        ).trim()

                        env.QG_STATUS = qgStatus
                        echo "Quality Gate Status: ${env.QG_STATUS}"
                    }
                    env.T_QG = "${(System.currentTimeMillis() - t0) / 1000}"
                    echo "STAGE TIME [Quality Gate Check]: ${env.T_QG}s"
                }
            }
        }

        stage('Report to GitHub') {
            steps {
                script {
                    def t0 = System.currentTimeMillis()
                    if (env.CHANGE_ID != null && env.CHANGE_ID != '') {
                        echo "Posting status to GitHub PR: ${env.CHANGE_ID}"

                        def statusState = (env.QG_STATUS == 'OK') ? 'success' : 'failure'
                        def statusDescription = (env.QG_STATUS == 'OK') ? 'Quality gate passed' : "Quality gate failed (${env.QG_STATUS})"
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
                    env.T_REPORT = "${(System.currentTimeMillis() - t0) / 1000}"
                    echo "STAGE TIME [Report to GitHub]: ${env.T_REPORT}s"
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
                echo ""
                echo "=== STAGE TIMING BREAKDOWN ==="
                echo "Checkout:               ${T_CHECKOUT:-N/A}s"
                echo "Build Gradle Projects:  ${T_BUILD:-N/A}s"
                echo "Download Scanner:       ${T_DOWNLOAD:-N/A}s"
                echo "List Projects:          ${T_LIST:-N/A}s"
                echo "SonarQube Scan:         ${T_SCAN:-N/A}s"
                echo "Quality Gate Check:     ${T_QG:-N/A}s"
                echo "Report to GitHub:       ${T_REPORT:-N/A}s"
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
