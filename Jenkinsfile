def sonarStatus = 'NOT CHECKED'
def COLOR_MAP = [
    'SUCCESS': '#36a64f',
    'FAILURE': '#FF0000',
    'UNSTABLE': '#FFFF00',
    'ABORTED': '#D3D3D3'
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
        NEXUS_GRP_REPO= 'vpro-maven-group'
        NEXUS_LOGIN   = 'nexuslogin'     // ID from Jenkins credentials
        SONARSERVER   = 'sonarserver'    // Jenkins-managed SonarQube server name
        SONARSCANNER  = 'sonarscanner'   // Jenkins tool name for Sonar Scanner
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
                scannerHome = tool SONARSCANNER
            }
            steps {
                withSonarQubeEnv(SONARSERVER) {
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
                        sonarStatus = qualityGate.status
                        echo "SonarQube Quality Gate: ${sonarStatus}"
                    }
                }
            }
        }

        stage("UploadArtifact") {
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
                        [
                            artifactId: 'vproapp',
                            classifier: '',
                            file: 'target/vprofile-v2.war',
                            type: 'war'
                        ]
                    ]
                )
            }
        }
    }

    post {
        always {
            echo 'Slack Notifications.'
            slackSend channel: '#jenkinscicd',
                color: COLOR_MAP[currentBuild.currentResult] ?: '#CCCCCC',
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build #${env.BUILD_NUMBER}\n" +
                         "*SonarQube Quality Gate:* ${sonarStatus}\n" +
                         "More info at: ${env.BUILD_URL}"
        }
    }
}
