def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger'
]

pipeline {
    agent any

    tools {
        maven 'MAVEN'
        jdk 'JDK17'
    }

    environment {
        SONAR_HOST_URL = 'http://192.168.1.105:9000'  // ✅ Use actual IP/host of SonarQube
        SONAR_PROJECT_KEY = 'vprofile'
        SONARSERVER = 'sonarserver'                 // From Jenkins tool config
        SONARSCANNER = 'sonarscanner'               // From Jenkins tool config
        NEXUSIP = '192.168.1.102'
        NEXUSPORT = '8081'
        RELEASE_REPO = 'vprofile-release'
        NEXUS_LOGIN = 'nexuslogin'
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn -s settings.xml -DskipTests install'
            }
            post {
                success {
                    echo "Now archiving"
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }

        stage('Test') {
            steps {
                sh 'mvn -s settings.xml test'
            }
        }

        stage('Checkstyle Analysis') {
            steps {
                sh 'mvn -s settings.xml checkstyle:checkstyle'
            }
        }

        stage('Sonar Analysis') {
            environment {
                scannerHome = tool "${SONARSCANNER}"
            }
            steps {
                withSonarQubeEnv("${SONARSERVER}") {
                    sh '''
                        ${scannerHome}/bin/sonar-scanner \
                          -Dsonar.projectKey=vprofile \
                          -Dsonar.projectName=vprofile \
                          -Dsonar.projectVersion=1.0 \
                          -Dsonar.sources=src/ \
                          -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                          -Dsonar.junit.reportsPath=target/surefire-reports/ \
                          -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                          -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
                    '''
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Fetch SonarQube Metrics') {
            steps {
                withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
                    script {
                        def sonarHost = "${SONAR_HOST_URL}"
                        def projectKey = "${SONAR_PROJECT_KEY}"

                        def getJson = { url ->
                            def response = sh(script: "curl -fsS -u $SONAR_TOKEN: '${url}'", returnStdout: true).trim()
                            return readJSON text: response
                        }

                        // Get Analysis ID
                        def analysisJson = getJson("${sonarHost}/api/project_analyses/search?project=${projectKey}")
                        def analysisId = analysisJson.analyses[0]?.key

                        if (!analysisId) {
                            error '❌ Failed to fetch analysis ID from SonarQube'
                        }

                        // Get Quality Gate Status
                        def qgStatusJson = getJson("${sonarHost}/api/qualitygates/project_status?analysisId=${analysisId}")
                        def qualityStatus = qgStatusJson.projectStatus.status

                        // Get Measures
                        def measuresJson = getJson("${sonarHost}/api/measures/component?component=${projectKey}&metricKeys=security_rating,reliability_rating,sqale_rating,bugs,vulnerabilities,code_smells,security_hotspots,coverage,duplicated_lines_density,ncloc")
                        def measures = measuresJson.component.measures

                        def get = { key -> measures.find { it.metric == key }?.value ?: 'N/A' }

                        // Prepare Slack message
                        env.SONAR_METRICS_MESSAGE = """
*SonarQube Report for ${projectKey}:*
✅ Quality Gate: *${qualityStatus}*
📦 Bugs: *${get('bugs')}*
🛡 Vulnerabilities: *${get('vulnerabilities')}*
🧹 Code Smells: *${get('code_smells')}*
🔥 Hotspots: *${get('security_hotspots')}*
🔐 Security Rating: *${get('security_rating')}*
🔧 Maintainability Rating: *${get('sqale_rating')}*
🐞 Reliability Rating: *${get('reliability_rating')}*
🧪 Coverage: *${get('coverage')}%*
🔁 Duplications: *${get('duplicated_lines_density')}%*
📄 Lines of Code: *${get('ncloc')}*
                        """.stripIndent().trim()

                        if (qualityStatus != "OK") {
                            error "❌ Quality Gate failed"
                        }
                    }
                }
            }
        }

        stage("Upload Artifact") {
            steps {
                nexusArtifactUploader(
                  nexusVersion: 'nexus3',
                  protocol: 'http',
                  nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                  groupId: 'QA',
                  version: "${env.BUILD_ID}.${env.BUILD_TAG}",
                  repository: "${RELEASE_REPO}",
                  credentialsId: "${NEXUS_LOGIN}",
                  artifacts: [[
                      artifactId: 'vproapp',
                      classifier: '',
                      file: 'target/vprofile-v2.war',
                      type: 'war'
                  ]]
                )
            }
        }
    }

    post {
        always {
            script {
                slackSend channel: '#jenkinscicd',
                    color: COLOR_MAP[currentBuild.currentResult],
                    tokenCredentialId: 'slacktoken',
                    message: """
*${currentBuild.currentResult}:* Job *${env.JOB_NAME}* Build *#${env.BUILD_NUMBER}*
🔗 <${env.BUILD_URL}|View Console Output>

${env.SONAR_METRICS_MESSAGE ?: '_No SonarQube data found._'}
                    """.stripIndent().trim()
            }
        }
    }
}
