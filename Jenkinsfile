
pipeline {
    agent any
    tools {
        maven "MAVEN"
        jdk "JDK17"
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
        sonarUrl = '192.168.1.102'
        SONAR_TOKEN = 'squ_90baa3a7fb3a092ce6d848ed90d37a6a32668c2b'
    }

    stages {
        stage('Build'){
            steps {
                sh 'mvn -s settings.xml -DskipTests install'
            }
            post {
                success {
                    echo "Now Archiving."
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }

        stage('Test'){
            steps {
                sh 'mvn -s settings.xml test'
            }

        }

        stage('Checkstyle Analysis'){
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
                   sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
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

        stage("Quality Gate") {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                    // true = set pipeline to UNSTABLE, false = don't
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("UploadArtifact"){
            steps{
                nexusArtifactUploader(
                  nexusVersion: 'nexus3',
                  protocol: 'http',
                  nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                  groupId: 'QA',
                  version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
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

    }
    post {
        always {
        script {
            def COLOR_MAP = [
                'SUCCESS': 'good',
                'FAILURE': 'danger',
                'UNSTABLE': 'warning'
            ]

            // Basic build result color
            def buildColor = COLOR_MAP.get(currentBuild.currentResult, 'danger')

            // SonarQube info
            def sonarToken = env.SONAR_TOKEN  // Make sure this is set as a Jenkins secret credential env var
            def sonarUrl = "http://192.168.1.102" // Update to your SonarQube URL
            def projectKey = "vprofile"         // Your SonarQube project key

            def getJson = { url ->
                sh(script: "curl -fsS -u ${sonarToken}: '${url}'", returnStdout: true).trim()
            }

            try {
                // Fetch latest analysisId
                def analysisId = ''
                for (int i = 0; i < 5; i++) {
                    def analysesJson = getJson("${sonarUrl}/api/project_analyses/search?project=${projectKey}")
                    def analyses = readJSON text: analysesJson
                    analysisId = analyses?.analyses?.getAt(0)?.key
                    if (analysisId) break
                    sleep 10
                }
                if (!analysisId) {
                    echo "⚠️ Could not fetch SonarQube analysis ID"
                }

                // Fetch Quality Gate status
                def qgJson = getJson("${sonarUrl}/api/qualitygates/project_status?analysisId=${analysisId}")
                def qgStatus = readJSON(text: qgJson).projectStatus.status

                // Fetch key measures
                def measuresJson = getJson("${sonarUrl}/api/measures/component?component=${projectKey}&metricKeys=bugs,code_smells,vulnerabilities")
                def measures = readJSON text: measuresJson

                def getMetricValue = { key ->
                    def m = measures.component.measures.find { it.metric == key }
                    return m ? m.value : "0"
                }

                def bugs = getMetricValue("bugs")
                def codeSmells = getMetricValue("code_smells")
                def vulnerabilities = getMetricValue("vulnerabilities")

                // Compose Slack message
                def slackMessage = """*SonarQube Quality Gate:* ${qgStatus}
*Bugs:* ${bugs}
*Code Smells:* ${codeSmells}
*Vulnerabilities:* ${vulnerabilities}
*Build Result:* ${currentBuild.currentResult}
Job: ${env.JOB_NAME} #${env.BUILD_NUMBER}
<${env.BUILD_URL}|Open Build>
"""

                // Send Slack message
                slackSend channel: '#jenkinscicd', color: buildColor, message: slackMessage

            } catch (Exception e) {
                echo "Error fetching SonarQube data or sending Slack message: ${e}"
                // Send a simpler Slack message fallback
                slackSend channel: '#jenkinscicd',
                          color: buildColor,
                          message: "*Build ${currentBuild.currentResult}* for job ${env.JOB_NAME} #${env.BUILD_NUMBER} \n<${env.BUILD_URL}|Open Build>"
            }
        }
    }
    }
}