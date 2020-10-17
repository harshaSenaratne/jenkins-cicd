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
def summary = ""

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
        stage('checking build type') {
            steps {
                sh 'printenv'
                script {
                    /* ---------------- PR requests section [just build and send build status]--------------------*/
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

                        /* ---------------- release requests [ get the approval ,checkout and release ] --------------------*/
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

                        /* ---------------- automatic release requests [ get the approval ,checkout and release ]--------------------*/
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

                        /* ---------------- branches create requests [ check , create and inform general]--------------------*/
                    } else if (env.JOB_BASE_NAME == 'onrequest-release' && env.par1 == 'create-release') {
                        // create release branch with tag
                        TYPE = "CREATE_RELEASE_BR"
                    } else if (env.JOB_BASE_NAME == 'onrequest-release' && env.par1 == 'create-hotfix') {
                        // create hotfix branch with tag. check whether still have one
                        TYPE = "CREATE_HOTFIX_BR"

                    } else {
                        echo "<<<could not find the change type>>>"
                    }

                    echo "type ==  $TYPE"
                }
            }
        }

        stage('setting variables') {
            steps {
                script {
                    BUILD_USER = currentBuild.getBuildCauses()[0].shortDescription
                    def commit = sh(returnStdout: true, script: 'git rev-parse HEAD')
                    COMMIT_AUTHOR = sh(returnStdout: true, script: "git --no-pager show -s --format='%an' ${commit}").trim()
                    COMMIT_MSG = sh(returnStdout: true, script: "git log --format=%B -n 1  ${commit}").trim()
                    echo "BUILD_USER : ${BUILD_USER}\n"
                    echo "COMMIT_MSG: ${COMMIT_MSG}\n"
                    echo "COMMIT_AUTHOR: ${COMMIT_AUTHOR}\n"
                    echo "commit: ${commit}\n"
                }
            }

            post {
                failure {
                    slackSend channel: 'error', color: COLOR_MAP[currentBuild.currentResult], message: "setting variables error"
                }
            }
        }


        stage(" ") {
            parallel {
                stage('Branch Creation[Slack]') {
                    when {
                        expression { TYPE == "CREATE_RELEASE_BR" || TYPE == "CREATE_HOTFIX_BR" }
                    }
                    stages {
                        stage('checking build type111') {
                            steps {
                                echo 'Branch Creation'
                            }
                        }
                        stage('checking build type2222') {
                            steps {
                                echo 'Branch Creation'
                            }
                        }
                    }
                }
                stage("non branch") {
                    when {
                        expression { TYPE != "CREATE_RELEASE_BR" && TYPE != "CREATE_HOTFIX_BR" }
                    }
                    stages {

                        stage('checkout code when on request release') {
                            when {
                                expression { TYPE == "QA_RELEASE_REQ" || TYPE == "STAGE_RELEASE_REQ" || TYPE == "DEV_RELEASE_REQ" || TYPE == "PROD_RELEASE_REQ" || TYPE == "HOTFIX_QA_RELEASE_REQ" || TYPE == "HOTFIX_STAGING_RELEASE_REQ" }
                            }
                            steps {
                                echo 'clean ws and checkkout code '
                            }
                        }

                        stage('build ') {
                            steps {
                                script {
                                    try {
                                        sh "./gradlew clean build -x test -x check"
                                    } catch (exception) {
                                        errorReportToSlack(TYPE, "Build", exception)
                                        throw exception
                                    }
                                }
                            }
                        }

                        stage('Junit & Jacoco') {
                            steps('running junit') {
                                script {
                                    try {
                                        sh 'chmod +x gradlew'
                                        sh './gradlew test jacocoTestReport --no-daemon'
                                        // if in case tests fail then subsequent stages
                                        // will not run .but post block in this stage will run
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
                                                reportName           : 'Junit Report'
                                        ]
                                    }
                                    catch (exception) {
                                        echo "$exception"
                                        errorReportToSlack(TYPE, "Junit", exception)
                                        throw exception

                                    }
                                    finally {
                                        summary = junit testResults: '**/build/test-results/test/*.xml'
                                        echo "test >>> ${summary.getProperties()}"
                                    }
                                }
                            }
                        }

                        stage('Checkstyle') {
                            steps {
                                script {
                                    try {
                                        sh "./gradlew checkstyleMain checkstyleTest"
                                    } catch (exception) {
                                        errorReportToSlack(TYPE, "Checkstyle", exception)
                                        throw exception
                                    } finally {
                                        publishHTML target: [
                                                allowMissing         : false,
                                                alwaysLinkToLastBuild: false,
                                                keepAll              : true,
                                                reportDir            : "build/reports/checkstyle",
                                                reportFiles          : '**/*',
                                                reportName           : 'Checkstyle Report'
                                        ]
                                    }
                                }
                            }
                        }

                        stage('PMD') {
                            steps {
                                script {
                                    try {
                                        sh "./gradlew pmdmain pmdtest"
                                    } catch (exception) {
                                        errorReportToSlack(TYPE, "PMD", exception)
                                        throw exception
                                    } finally {
                                        publishHTML target: [
                                                allowMissing         : false,
                                                alwaysLinkToLastBuild: false,
                                                keepAll              : true,
                                                reportDir            : "build/reports/pmd",
                                                reportFiles          : 'main.html,test.html',
                                                reportName           : 'PMD Report'
                                        ]
                                    }
                                }
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
                                unstable {
                                    errorReportToSlack(TYPE, "SQ", "eroor")
                                }
                            }
                        }
                    }
                }

            }

        }
    }
}

def errorReportToSlack(TYPE, stage, errorInfo) {
    echo " type =$TYPE  info = $errorInfo  stage = $stage"
    if (TYPE == "CREATE_RELEASE_BR" || TYPE == "CREATE_HOTFIX_BR") {
        notifySlack("admin",errorInfo,TYPE)
    } else if (TYPE == "QA_RELEASE_REQ" || TYPE == "STAGE_RELEASE_REQ" || TYPE == "DEV_RELEASE_REQ" || TYPE == "PROD_RELEASE_REQ" || TYPE == "HOTFIX_QA_RELEASE_REQ" || TYPE == "HOTFIX_STAGING_RELEASE_REQ") {
        notifySlack("admin",errorInfo,TYPE)
    } else if (TYPE == "DEV_PR" || TYPE == "RELEASE_PR" || TYPE == "HOTFIX_PR" || TYPE == "PROD_PR" || TYPE == "HOTFIX_PROD_PR") {
        notifySlack("pull-request",errorInfo,TYPE)
    } else if (TYPE == "DEV_RELEASE" || TYPE == "QA_RELEASE" || TYPE == "PROD_RELEASE" || TYPE == "HOTFIX_QA_RELEASE") {
        notifySlack("error",errorInfo,TYPE)
    }
}

def notifySlack(channel, error, type) {
    withCredentials([string(credentialsId: 'slack-token', variable: 'st'), string(credentialsId: 'jen', variable: 'jenn')]) {
        script {
            sh "curl --location --request POST '$st'  --header 'Content-Type: application/json' --data-raw '{ \"channel\": \"${channel}\", \"text\": \"error=${error} : type= ${type} : stage=${stage}\"}'"
        }
    }

}
