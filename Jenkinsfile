def sonarStatus = 'NOT CHECKED'  // ✅ Declare at global level

def COLOR_MAP = [
    'SUCCESS': '#36a64f',   // green
    'FAILURE': '#FF0000',   // red
    'UNSTABLE': '#FFFF00',  // yellow
    'ABORTED': '#D3D3D3'    // gray
]

pipeline {
    agent any

    environment {
        scannerHome = tool "${SONARSCANNER}"
    }

    stages {
        stage('Sonar Analysis') {
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

        stage("Quality Gate") {
            steps {
                script {
                    timeout(time: 1, unit: 'HOURS') {
                        def qualityGate = waitForQualityGate(abortPipeline: true)
                        sonarStatus = qualityGate.status  // ✅ Modify global variable
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Slack Notifications.'
            slackSend channel: '#jenkinscicd',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}\n" +
                         "*SonarQube Quality Gate:* ${sonarStatus}\n" +
                         "More info at: ${env.BUILD_URL}"
        }
    }
}
