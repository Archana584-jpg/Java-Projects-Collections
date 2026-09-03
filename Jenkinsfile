pipeline {
    agent {
        docker {
            image 'maven:3.8.6-eclipse-temurin-11'
            args '-v /root/.m2:/root/.m2'
        }
    }

    environment {
        GITHUB_TOKEN = credentials('archana-sonar')
        SONAR_TOKEN = credentials('sonar-token')
        SONAR_HOST_URL = 'http://15.206.213.78:9000'
        PROJECT_KEY = 'java-projects-collections'
        GIT_BRANCH = "${env.CHANGE_BRANCH ?: env.GIT_BRANCH}"
    }

    options {
        timeout(time: 15, unit: 'MINUTES')
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
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile -DskipTests'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Scan') {
            when {
                expression { env.CHANGE_ID != null }
            }
            steps {
                sh '''
                    mvn sonar:sonar \
                        -Dsonar.projectKey=${PROJECT_KEY} \
                        -Dsonar.host.url=${SONAR_HOST_URL} \
                        -Dsonar.login=${SONAR_TOKEN} \
                        -Dsonar.pullrequest.key=${CHANGE_ID} \
                        -Dsonar.pullrequest.branch=${CHANGE_BRANCH} \
                        -Dsonar.pullrequest.base=${CHANGE_TARGET} \
                        -Dsonar.projectVersion=${BUILD_NUMBER}
                '''
            }
        }

        stage('Quality Gate Check') {
            when {
                expression { env.CHANGE_ID != null }
            }
            steps {
                script {
                    timeout(time: 2, unit: 'MINUTES') {
                        def qgStatus = sh(
                            script: '''
                                for i in {1..12}; do
                                    STATUS=$(curl -s -u ${SONAR_TOKEN}: "${SONAR_HOST_URL}/api/qualitygates/project_status?projectKey=${PROJECT_KEY}" | grep -o '"status":"[^"]*' | cut -d'"' -f4)
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
                    def statusState = (env.QG_STATUS == 'OK' || env.QG_STATUS == 'PASSED') ? 'success' : 'failure'
                    def statusDescription = (env.QG_STATUS == 'OK' || env.QG_STATUS == 'PASSED') ? 'Quality gate passed' : "Quality gate failed (${env.QG_STATUS})"
                    def targetUrl = "${env.SONAR_HOST_URL}/dashboard?id=${PROJECT_KEY}&pullRequest=${CHANGE_ID}"

                    sh '''
                        curl -X POST \
                            -H "Authorization: token ${GITHUB_TOKEN}" \
                            -H "Content-Type: application/json" \
                            -d '{
                                "state": "''' + statusState + '''",
                                "description": "''' + statusDescription + '''",
                                "target_url": "''' + targetUrl + '''",
                                "context": "Jenkins/SonarQube"
                            }' \
                            ${REPO_URL}/statuses/${COMMIT_HASH}
                    '''
                }
            }
        }
    }

    post {
        always {
            junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
        }
        failure {
            script {
                if (env.CHANGE_ID != null) {
                    sh '''
                        curl -X POST \
                            -H "Authorization: token ${GITHUB_TOKEN}" \
                            -H "Content-Type: application/json" \
                            -d '{
                                "state": "error",
                                "description": "Build failed",
                                "target_url": "${BUILD_URL}",
                                "context": "Jenkins/Build"
                            }' \
                            ${REPO_URL}/statuses/${COMMIT_HASH}
                    '''
                }
            }
        }
    }
}
