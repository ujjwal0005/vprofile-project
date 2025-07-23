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
        SNAP_REPO     = 'vprofile-snapshot'
        NEXUS_USER    = 'admin'
        NEXUS_PASS    = 'admin123'
        RELEASE_REPO  = 'vprofile-release'
        CENTRAL_REPO  = 'vpro-maven-central'
        NEXUSIP       = '192.168.1.102'
        NEXUSPORT     = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group'
        NEXUS_LOGIN   = 'nexuslogin'
        SONARSERVER   = 'sonarserver'
        SONARSCANNER  = 'sonarscanner'
        SONAR_HOST_URL = 'http://192.168.1.101:9000' // Replace with your SonarQube URL
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn -s settings.xml -DskipTests install'
            }
            post {
                success {
                    echo "Archiving WAR file"
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

        stage('SonarQube Analysis') {
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

        stage("Quality Gate & Sonar Report to Slack") {
            steps {
                script {
                    timeout(time: 1, unit: 'MINUTES') {
                        def qualityGate = waitForQualityGate()
                        def sonarHost = "${env.SONAR_HOST_URL}"
                        def projectKey = 'vprofile'
                        def authToken = credentials('sonar-token') // Must be created in Jenkins Credentials

                        def metrics = ['bugs', 'vulnerabilities', 'code_smells', 'coverage', 'duplicated_lines_density'].join(',')

                        def response = httpRequest(
                            url: "${sonarHost}/api/measures/component?component=${projectKey}&metricKeys=${metrics}",
                            authentication: "${authToken}",
                            httpMode: 'GET',
                            contentType: 'APPLICATION_JSON'
                        )

                        def json = readJSON text: response.content
                        def measures = json.component.measures.collectEntries {
                            [(it.metric): it.value]
                        }

                        def resultColor = (qualityGate.status == 'OK') ? 'good' : 'danger'
                        def analysisResult = """
*SonarQube Quality Gate Result:* *${qualityGate.status}*
>*Coverage:* ${measures.coverage ?: 'N/A'}%
>*Bugs:* ${measures.bugs ?: 'N/A'}
>*Vulnerabilities:* ${measures.vulnerabilities ?: 'N/A'}
>*Code Smells:* ${measures.code_smells ?: 'N/A'}
>*Duplicated Lines:* ${measures.duplicated_lines_density ?: 'N/A'}%
<${env.BUILD_URL}|Open Jenkins Build>
"""

                        // Send to #codeanalysis channel
                        slackSend(
                            channel: '#codeanalysis',
                            color: resultColor,
                            message: analysisResult
                        )

                        // Also fail the pipeline if Quality Gate failed
                        if (qualityGate.status != 'OK') {
                            error("SonarQube Quality Gate FAILED")
                        }
                    }
                }
            }
        }

        stage("Upload Artifact to Nexus") {
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
            echo 'Sending Slack summary notification.'
            slackSend(
                channel: '#jenkinscicd',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job `${env.JOB_NAME}` build #${env.BUILD_NUMBER}\n<${env.BUILD_URL}|Click to View>"
            )
        }
    }
}
