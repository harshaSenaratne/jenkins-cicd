#!groovy
def COLOR_MAP = ['SUCCESS': 'good', 'FAILURE': 'danger',]

enum Type {
    DEV_PR,
    RELEASE_PR,
    HOTFIX_PR,
    PROD_PR,
    HOTFIX_PROD_PR,
    QA_RELEASE_REQ,
    STAGE_RELEASE_REQ,
    DEV_RELEASE_REQ,
    PROD_RELEASE_REQ,
    HOTFIX_QA_RELEASE_REQ,
    HOTFIX_STAGING_RELEASE_REQ,
    CREATE_RELEASE_BR,
    CREATE_HOTFIX_BR,
    DEV_RELEASE,
    QA_RELEASE,
    PROD_RELEASE,
    HOTFIX_QA_RELEASE
}

def COMMIT_AUTHOR = ""
def BUILD_USER = ""
def COMMIT_MSG = ""
def TYPE = ""


//TODO chnageset  ,  changelog, try catch bloc , send test summary, sonar summary ,
pipeline {
//    try{
    agent any
    options {
//        skipDefaultCheckout()
        buildDiscarder(logRotator(numToKeepStr: '1'))
    }

    // =============== stages====================
    stages {
        stage('identifying build type') {
            steps {
                sh 'printenv'
                script {
                    if (env.JOB_BASE_NAME.startsWith('PR') && env.CHANGE_TARGET == "develop") {
                        TYPE = "DEV_PR"
                    } else if (env.JOB_BASE_NAME.startsWith('PR') && env.CHANGE_TARGET.startsWith('release')) {
                        //release bug fixing PR
                        TYPE = "RELEASE_PR"
                    } else if (env.JOB_BASE_NAME.startsWith('PR') && env.CHANGE_TARGET.startsWith('hotfix')) {
                        //hotfix fixing PR
                        TYPE = "HOTFIX_PR"
                    } else if (env.JOB_BASE_NAME.startsWith('PR') && env.CHANGE_TARGET.startsWith('master') && env.CHANGE_BRANCH.startsWith('release')) {
                        //prod release PR
                        TYPE = "PROD_PR"
                    } else if (env.JOB_BASE_NAME.startsWith('PR') && env.CHANGE_TARGET.startsWith('master') && env.CHANGE_BRANCH.startsWith('hotfix')) {
                        // hotfix prod release PR
                        TYPE = "HOTFIX_PROD_PR"
                    } else if (env.JOB_BASE_NAME == 'onrequest-release' && env.par1 == 'qa') {
                        // qa release request
                        TYPE = "QA_RELEASE_REQ"
                    } else if (env.JOB_BASE_NAME == 'onrequest-release' && env.par1 == 'staging') {
                        // prep release request
                        TYPE = "STAGE_RELEASE_REQ"
                    } else if (env.JOB_BASE_NAME == 'onrequest-release' && env.par1 == 'dev') {
                        // dev release request
                        TYPE = "DEV_RELEASE_REQ"
                    } else if (env.JOB_BASE_NAME == 'onrequest-release' && env.par1 == 'prod') {
                        // prod release request with LATEST tag
                        TYPE = "PROD_RELEASE_REQ"
                    } else if (env.JOB_BASE_NAME == 'onrequest-release' && env.par1 == 'hotfix-qa') {
                        // hotfix qa release request
                        TYPE = "HOTFIX_QA_RELEASE_REQ"
                    } else if (env.JOB_BASE_NAME == 'onrequest-release' && env.par1 == 'hotfix-staging') {
                        // hotfix staging release request
                        TYPE = "HOTFIX_STAGING_RELEASE_REQ"
                    } else if (env.JOB_BASE_NAME == 'onrequest-release' && env.par1 == 'create-release') {
                        // create release branch with tag
                        TYPE = "CREATE_RELEASE_BR"
                    } else if (env.JOB_BASE_NAME == 'onrequest-release' && env.par1 == 'create-hotfix') {
                        // create hotfix branch with tag. check whether still have one
                        TYPE = "CREATE_HOTFIX_BR"
                    } else if (env.JOB_BASE_NAME == "develop") {
                        // start dev release. no need approval .
                        TYPE = "DEV_RELEASE"
                    } else if (env.JOB_BASE_NAME.startsWith('release')) {
                        // qa  release request. need approval
                        TYPE = "QA_RELEASE"
                    } else if (env.JOB_BASE_NAME.startsWith('master')) {
                        // tag and start prod"  release request .need approval.check last merged branch .if hot fix bumpup hotfix version
                        TYPE = "PROD_RELEASE"
                    } else if (env.JOB_BASE_NAME.startsWith('hotfix')) {
                        //  qa  release request. need approval
                        TYPE = "HOTFIX_QA_RELEASE"
                    } else {
                        echo "<<<could not find the change type>>>"
                    }

                    echo "type ==  $TYPE"
                }
            }
        }
        stage('getting variable values ') {
            steps {
                script {
                    BUILD_USER = currentBuild.getBuildCauses()[0].shortDescription
                    def commit = sh(returnStdout: true, script: 'git rev-parse HEAD')
                    COMMIT_AUTHOR = sh(returnStdout: true, script: "git --no-pager show -s --format='%an' ${commit}").trim()
                    COMMIT_MSG = sh(returnStdout: true, script: "git log --format=%B -n 1  ${commit}").trim()
                    echo "Current build was caused by: ${BUILD_USER}\n"
                    echo "COMMIT_MSG: ${COMMIT_MSG}\n"
                }
            }

            post {
                failure {
                    echo 'getting variable values  error'
                    slackSend channel: 'error',
                            color: COLOR_MAP[currentBuild.currentResult],
                            message: " ${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} by ${BUILD_USER}\n More info at: ${env.BUILD_URL}"

                }
            }
        }
        stage('build ') {
            steps {
//                notifySlack()
                sh "./gradlew clean build -x test -x check"
            }

            post {
                failure {
                    echo 'build  error'
                    slackSend channel: 'error',
                            color: COLOR_MAP[currentBuild.currentResult],
                            message: " ${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} by ${BUILD_USER}\n More info at: ${env.BUILD_URL}"

                }
            }
        }

        stage('test') {
            steps {
                script {
                    try {
                        sh 'chmod +x gradlew'
                        sh './gradlew test jacocoTestReport --no-daemon'
                        // if in case tests fail then subsequent stages
                        // will not run .but post block in this stage will run
                    } catch (exception) {
                        echo "$exception"

                    } finally {
                        junit '**/build/test-results/test/*.xml'
                    }
                }
            }
            post {
                success {
                    echo "printing this message even unit test ara failing"
                    step([$class          : 'JacocoPublisher',
                          execPattern     : '**/build/jacoco/*.exec',
                          classPattern    : '**/build/classes',
                          sourcePattern   : 'src/main/java',
                          exclusionPattern: 'src/test*'
                    ])
                    publishHTML target: [
                            allowMissing         : false,
                            alwaysLinkToLastBuild: false,
                            keepAll              : true,
                            reportDir            : "build/reports/tests/test",
                            reportFiles          : 'index.html',
                            reportName           : 'HTML Report'
                    ]
                }
                unstable {
                    echo "printing this message even unit test ara unstable"
                }
                failure {
                    echo 'test error'
                    slackSend channel: 'error', color: COLOR_MAP[currentBuild.currentResult], message: "junit error"

                }
            }
        }

        stage('Checkstyle') {
            steps {
                sh "./gradlew checkstyleMain checkstyleTest"
                recordIssues(tools: [checkStyle(reportEncoding: 'UTF-8')])
            }
        }


        stage('SQ analysis') { //there are 2 ways to configure sonar in jenkins
            //one method usingg jenkins global configuration
            steps {
                script {
//                    def scannerHome = tool 'SonarScanner 4.0';
//                    withSonarQubeEnv('mysona') { // If you have configured more than one global server connection, you can specify its name
//                        sh "${scannerHome}/bin/sonar-scanner"
//                    }

                    //other one is using gradle build
                    withSonarQubeEnv() { // Will pick the global server connection you have configured
                        sh "./gradlew sonarqube -Dsonar.projectName=${TYPE}"
                    }
                    timeout(time: 1, unit: 'HOURS') {
                        // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                        // true = set pipeline to UNSTABLE, false = don't
                        waitForQualityGate abortPipeline: true
                    }
                }
            }

            post {
                failure {
                    echo 'Sonarqube error'
                    slackSend channel: 'error',
                            color: COLOR_MAP[currentBuild.currentResult],
                            message: "Sonarqube error"

                }
            }
        }
//
//        stage('getting approval for qa release') {
//            when { branch 'develop' }
//            steps {
//                echo 'getting approval for qa release'
//                slackSend channel: 'qa-release-approval',
//                        color: 'good',
//                        message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} by ${BUILD_USER}\n More info at: ${env.BUILD_URL}"
//
//            }
//        }
//
        stage('inform  build status to slack ') {
            steps {
                echo 'inform  build status to slack'
                slackSend channel: 'general', color: COLOR_MAP[currentBuild.currentResult], message: "build completed"

            }
        }
    }
}

def notifySlack() {
    withCredentials([string(credentialsId: 'slack-token', variable: 'st'), string(credentialsId: 'jen', variable: 'jenn')]) {
        script {
            sh "curl --location --request POST '$st'  --header 'Content-Type: application/json' --data-raw '{\n" +
                    "\"blocks\": [\n" +
                    "{\n" +
                    "\"type\": \"section\",\n" +
                    "\"text\": {\n" +
                    "\"type\": \"mrkdwn\",\n" +
                    "\"text\": \"Danny Torrence left the following review for your property:\"\n" +
                    "}\n" +
                    "},\n" +
                    "{\n" +
                    "\"type\": \"section\",\n" +
                    "\"block_id\": \"section567\",\n" +
                    "\"text\": {\n" +
                    "\"type\": \"mrkdwn\",\n" +
                    "\"text\": \"<https://example.com|Overlook Hotel> \\n :star: \\n Doors had too many axe holes, guest in room 237 was far too rowdy, whole place felt stuck in the 1920s.\"\n" +
                    "},\n" +
                    "\"accessory\": {\n" +
                    "\"type\": \"image\",\n" +
                    "\"image_url\": \"https://is5-ssl.mzstatic.com/image/thumb/Purple3/v4/d3/72/5c/d3725c8f-c642-5d69-1904-aa36e4297885/source/256x256bb.jpg\",\n" +
                    "\"alt_text\": \"Haunted hotel image\"\n" +
                    "}\n" +
                    "},\n" +
                    "{\n" +
                    "\"type\": \"section\",\n" +
                    "\"block_id\": \"section789\",\n" +
                    "\"fields\": [\n" +
                    "{\n" +
                    "\"type\": \"mrkdwn\",\n" +
                    "\"text\": \"*Average Rating*\\n1.0\"\n" +
                    "}\n" +
                    "]\n" +
                    "},\n" +
                    "{\n" +
                    "\"type\": \"actions\",\n" +
                    "\"elements\": [\n" +
                    "{\n" +
                    "\"type\": \"button\",\n" +
                    "\"text\": {\n" +
                    "\"type\": \"plain_text\",\n" +
                    "\"text\": \"Reply to review\",\n" +
                    "\"emoji\": false\n" +
                    "}\n" +
                    "}\n" +
                    "]\n" +
                    "},\n" +
                    "{\n" +
                    "\"type\": \"section\",\n" +
                    "\"text\": {\n" +
                    "\"type\": \"mrkdwn\",\n" +
                    "\"text\": \"This is a section block with a button.\"\n" +
                    "},\n" +
                    "\"accessory\": {\n" +
                    "\"type\": \"button\",\n" +
                    "\"text\": {\n" +
                    "\"type\": \"plain_text\",\n" +
                    "\"text\": \"Click Me\",\n" +
                    "\"emoji\": true\n" +
                    "},\n" +
                    "\"value\": \"click_me_123\",\n" +
                    "\"url\": \"https://google.com\",\n" +
                    "\"action_id\": \"button-action\"\n" +
                    "}\n" +
                    "},\n" +
                    "{\n" +
                    "\"type\": \"section\",\n" +
                    "\"text\": {\n" +
                    "\"type\": \"plain_text\",\n" +
                    "\"text\": \"This is a plain text section block.\",\n" +
                    "\"emoji\": true\n" +
                    "}\n" +
                    "}\n" +
                    "]\n" +
                    "}'"
        }
    }

}
