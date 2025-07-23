def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
    'UNSTABLE': 'warning'
]

pipeline {
    agent any

    tools {
        maven 'MAVEN'
        jdk 'JDK17'
    }

    environment {
        SNAP_REPO = 'vprofile-snapshot'
        NEXUS_USER = 'admin'
        NEXUS_PASS = 'admin123'
        RELEASE_REPO = 'vprofile-release'
        CENTRAL_REPO = 'vpro-maven-central'
        NEXUSIP = '192.168.1.102'
        NEXUSPORT = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group'
        NEXUS_LOGIN = 'nexuslogin'
        SONARSERVER = 'sonarserver'
        SONARSCANNER = 'sonarscanner'
        SONAR_TOKEN = credentials('sonartoken')
        SONAR_PROJECT_KEY = 'vprofile'
        SONAR_HOST_URL = 'http://sonarurl'
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
                    sh '''${scannerHome}/bin/sonar-scanner \
                    -Dsonar.projectKey=vprofile \
                    -Dsonar.projectName=vprofile \
                    -Dsonar.projectVersion=1.0 \
                    -Dsonar.sources=src/ \
                    -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                    -Dsonar.junit.reportsPath=target/surefire-reports/ \
                    -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                    -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Upload Artifact') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                    groupId: 'QA',
                    version: "${env.BUILD_ID}.${env.BUILD_TAG}",
                    repository: "${RELEASE_REPO}",
                    credentialsId: "${NEXUS_LOGIN}",
                    artifacts: [
                        [artifactId: 'vproapp',
                         classifier: '',
                         file: 'target/vprofile-v2.war',
                         type: 'war']
                    ]
                )
            }
        }

        stage('SonarQube Summary to Slack') {
            steps {
                script {
                    def shTrimmed = { cmd -> sh(script: cmd, returnStdout: true).trim() }

                    def analysisId = ''
                    for (int i = 0; i < 10; i++) {
                        analysisId = shTrimmed("""
                          curl -fsS -u ${SONAR_TOKEN}: \
                            '${SONAR_HOST_URL}/api/project_analyses/search?project=${SONAR_PROJECT_KEY}' \
                            | jq -r '.analyses[0].key'
                        """)
                        if (analysisId && analysisId != 'null') {
                            echo "✅ Found analysis ID: ${analysisId}"
                            break
                        } else {
                            echo "⏳ Attempt ${i+1}: Analysis ID not found. Retrying in 10s..."
                            sleep 10
                        }
                    }
                    if (!analysisId || analysisId == 'null') {
                        error '❌ Could not find Analysis ID after retries!'
                    }

                    def statusJson = shTrimmed("""
                      curl -fsS -u ${SONAR_TOKEN}: \
                        '${SONAR_HOST_URL}/api/qualitygates/project_status?analysisId=${analysisId}'
                    """)
                    def statusObj = readJSON text: statusJson
                    def qualityGateStatus = statusObj.projectStatus.status

                    def measuresJson = shTrimmed("""
                      curl -fsS -u ${SONAR_TOKEN}: \
                        '${SONAR_HOST_URL}/api/measures/component?component=${SONAR_PROJECT_KEY}&metricKeys=security_rating,reliability_rating,sqale_rating,bugs,vulnerabilities,code_smells,coverage,duplicated_lines_density,ncloc'
                    """)
                    def measures = readJSON text: measuresJson

                    def getMetricValue = { key ->
                        def metric = measures.component.measures.find { it.metric == key }
                        return metric ? metric.value : 'N/A'
                    }

                    def metrics = [
                        "🔐 Security Rating: ${getMetricValue('security_rating')}",
                        "🐛 Reliability Rating: ${getMetricValue('reliability_rating')}",
                        "🛠️ Maintainability: ${getMetricValue('sqale_rating')}",
                        "🧪 Bugs: ${getMetricValue('bugs')}",
                        "🚨 Vulnerabilities: ${getMetricValue('vulnerabilities')}",
                        "📦 Code Smells: ${getMetricValue('code_smells')}",
                        "📊 Coverage: ${getMetricValue('coverage')}%",
                        "🔁 Duplications: ${getMetricValue('duplicated_lines_density')}%",
                        "📏 LOC: ${getMetricValue('ncloc')}"
                    ].join('\n')

                    slackSend(
                        channel: '#jenkinscicd',
                        color: COLOR_MAP[currentBuild.currentResult],
                        message: """
*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build #${env.BUILD_NUMBER}
📌 Quality Gate: *${qualityGateStatus}*
${metrics}
🔗 <${env.BUILD_URL}|Open Build>
"""
                    )
                }
            }
        }
    }

    post {
        failure {
            slackSend channel: '#jenkinscicd',
                color: 'danger',
                message: "*FAILURE:* Job ${env.JOB_NAME} build #${env.BUILD_NUMBER} failed.\nMore info: ${env.BUILD_URL}"
        }
        success {
            echo "✅ Pipeline completed successfully!"
        }
    }
}
