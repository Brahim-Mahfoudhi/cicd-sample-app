pipeline {
    agent { label 'App' }
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        JENKINS_SERVER = 'http://172.16.128.100:8080' //NEEDS TO BE CHANGED
        DOTNET_PROJECT_PATH = dotnet-2425-tiao1/Rise.Server/Rise.Server.csproj'
        DOTNET_TEST_PATH = 'dotnet-2425-tiao1/Rise.Domain.Tests/Rise.Domain.Tests.csproj'
        PUBLISH_OUTPUT = 'publish'
        DOTNET_ENVIRONMENT = 'Production'
        DOTNET_CONNECTION_STRING = 'Server=localhost,1433;Database=SportStore;User Id=sa;Password=Drgnnrblnc19;Trusted_Connection=False;MultipleActiveResultSets=True;' // NEEDS TO BE CHANGED
        DISCORD_WEBHOOK_URL = "https://discord.com/api/webhooks/1301160382307766292/kROxjtgZ-XVOibckTMri2fy5-nNOEjzjPLbT9jEpr_R0UH9JG0ZXb2XzUsYGE0d3yk6I"
        JENKINS_CREDENTIALS_ID = "jenkins-master-key"
        SSH_KEY_FILE = '/var/lib/jenkins/.ssh/id_rsa'
        REMOTE_HOST = 'jenkins@172.16.128.101' // NEEDS TO BE CHANGED
        COVERAGE_REPORT_PATH = '/var/lib/jenkins/agent/workspace/dotnet_pipeline/coverage/coverage.cobertura.xml'
        COVERAGE_REPORT_DIR = '/var/lib/jenkins/agent/workspace/dotnet_pipeline/coverage-report/'
        TRX_FILE_PATH = 'dotnet-2425-tiao1/Rise.Domain.Tests/TestResults/test-results.trx'
        TEST_RESULT_PATH = 'dotnet-2425-tiao1/Rise.Domain.Tests/TestResults'
        TRX_TO_XML_PATH = 'dotnet-2425-tiao1/Rise.Domain.Tests/TestResults/test-results.xml'
        PUBLISH_DIR_PATH = '/var/lib/jenkins/artifacts/'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                script {
                    git url: 'https://github.com/HOGENT-RISE/dotnet-2425-tiao1.git'
                    echo 'Gather GitHub info!'
                    def gitInfo = sh(script: 'git show -s HEAD --pretty=format:"%an%n%ae%n%s%n%H%n%h" 2>/dev/null', returnStdout: true).trim().split("\n")
                    env.GIT_AUTHOR_NAME = gitInfo[0]
                    env.GIT_AUTHOR_EMAIL = gitInfo[1]
                    env.GIT_COMMIT_MESSAGE = gitInfo[2]
                    env.GIT_COMMIT = gitInfo[3]
                    env.GIT_BRANCH = gitInfo[4]
                }
            }
        }

        stage('Restore Dependencies') {
            steps {
                sh "dotnet restore ${DOTNET_PROJECT_PATH}"
                sh "dotnet restore ${DOTNET_TEST_PATH}"
            }
        }

        stage('Build Application') {
            steps {
                sh "dotnet build ${DOTNET_PROJECT_PATH}"
            }
        }

        stage('Running Unit Tests') {
            steps {
                sh "dotnet test ${DOTNET_TEST_PATH} --logger 'trx;LogFileName=test-results.trx' /p:CollectCoverage=true /p:CoverletOutput=${COVERAGE_REPORT_PATH} /p:CoverletOutputFormat=cobertura"
            }
        }

        stage('Coverage Report') {
            steps {
                echo 'Generating code coverage report...'
                script {
                    sh "/home/jenkins/.dotnet/tools/reportgenerator -reports:${COVERAGE_REPORT_PATH} -targetdir:${COVERAGE_REPORT_DIR} -reporttypes:Html"
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: true, reportDir: COVERAGE_REPORT_DIR, reportFiles: 'index.html', reportName: 'Coverage Report'])
                }
            }
        }

        stage('Publish Application') {
            steps {
                sh "dotnet publish ${DOTNET_PROJECT_PATH} -c Release -o ${PUBLISH_OUTPUT}"
            }
        }

        stage('Deploy to Remote Server') {
            steps {
                sshagent([JENKINS_CREDENTIALS_ID]) {
                    script {
                        sh """
                            scp -i ${SSH_KEY_FILE} -o StrictHostKeyChecking=no -r ${PUBLISH_OUTPUT}/* ${REMOTE_HOST}:${PUBLISH_DIR_PATH}
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Build and deployment completed successfully!'
            archiveArtifacts artifacts: '**/*.dll', fingerprint: true
            archiveArtifacts artifacts: "${TRX_FILE_PATH}", fingerprint: true
            script {
                sendDiscordNotification("Build Success")
            }
        }
        failure {
            echo 'Build or deployment has failed.'
            script {
                sendDiscordNotification("Build Failed")
            }
        }
        always {
            echo 'Build process has completed.'
            echo 'Generate Test report...'
            sh "/home/jenkins/.dotnet/tools/trx2junit --output ${TEST_RESULT_PATH} ${TRX_FILE_PATH}"
            junit "${TRX_TO_XML_PATH}"
        }
    }
}

def sendDiscordNotification(status) {
    script {
        discordSend(
            title: "${env.JOB_NAME} - ${status}",
            description: """
                Build #${env.BUILD_NUMBER} ${status == "Build Success" ? 'completed successfully!' : 'has failed!'}
                **Commit**: ${env.GIT_COMMIT}
                **Author**: ${env.GIT_AUTHOR_NAME} <${env.GIT_AUTHOR_EMAIL}>
                **Branch**: ${env.GIT_BRANCH}
                **Message**: ${env.GIT_COMMIT_MESSAGE}
                
                [**Build output**](${JENKINS_SERVER}/job/${env.JOB_NAME}/${env.BUILD_NUMBER}/console) - Build output
                [**Test result**](${JENKINS_SERVER}/job/dotnet_pipeline/lastBuild/testReport/) - Test result
                [**Coverage report**](${JENKINS_SERVER}/job/dotnet_pipeline/lastBuild/Coverage_20Report/) - Coverage report
                [**History**](${JENKINS_SERVER}/job/dotnet_pipeline/${env.BUILD_NUMBER}/testReport/history/) - History
            """,
            footer: "Build Duration: ${currentBuild.durationString.replace(' and counting', '')}",
            webhookURL: DISCORD_WEBHOOK_URL,
            result: status == "Build Success" ? 'SUCCESS' : 'FAILURE'
        )
    }
}