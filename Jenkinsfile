#!groovy

import groovy.json.JsonSlurperClassic
node {

    BUILD_NUMBER = env.BUILD_NUMBER
    RUN_ARTIFACT_DIR = "tests/${BUILD_NUMBER}"
    SFDC_USERNAME = ''
    BRANCH_NAME = env.BRANCH_NAME

    HUB_ORG = 'HUB_ORG'
    //def SFDC_HOST = env.SFDC_HOST_DH
    //def JWT_KEY_CRED_ID = env.JWT_KEY_FILE
    JWT_KEY_CRED_ID = 'JWT_KEY_FILE'
    CONNECTED_APP_CONSUMER_KEY = 'CONNECTED_APP_CONSUMER_KEY'
    println('Is pr : ' + isPRMergeBuild())
    if (isPRMergeBuild()) {
        checkoutSource()
        createScratchOrg()
        pushSource()
        runApexTests()
        deleteScratchOrg()
    } else {
        checkoutSource()
        createScratchOrg()
        pushSource()
        deleteScratchOrg()
    }

}

def isPRMergeBuild() {
    println('Branch Name : '+ BRANCH_NAME)
    return (BRANCH_NAME.startsWith('PR-'))
}

def checkoutSource() {
    stage('checkout source') {
        checkout scm
    }
}
def createScratchOrg() {
    withCredentials([file(credentialsId: JWT_KEY_CRED_ID, variable: 'jwt_key_file'),
        string(credentialsId: HUB_ORG, variable: 'HUB'),
        string(credentialsId: CONNECTED_APP_CONSUMER_KEY, variable: 'CONNECTED_APP_KEY')
    ]) {
        stage('Create Scratch Org') {

            rc = sh returnStatus: true, script: "sfdx force:auth:jwt:grant --clientid ${CONNECTED_APP_KEY} --username ${HUB} --jwtkeyfile ${jwt_key_file} --setdefaultdevhubusername "
            if (rc != 0) {
                error 'hub org authorization failed'
            }

            // need to pull out assigned username
            rmsg = sh returnStdout: true, script: "sfdx force:org:create --definitionfile config/project-scratch-def.json --json --setdefaultusername"
            def beginIndex = rmsg.indexOf('{')
            def endIndex = rmsg.indexOf('}')
            def jsobSubstring = rmsg.substring(beginIndex)
            def jsonSlurperClass = new JsonSlurperClassic()
            def robj = jsonSlurperClass.parseText(jsobSubstring)
            SFDC_USERNAME = robj.result.username
            robj = null

        }
    }
}
def deleteScratchOrg() {
    stage('Delete Test Org') {
        timeout(time: 120, unit: 'SECONDS') {
            rc = sh returnStatus: true, script: "sfdx force:org:delete --targetusername ${SFDC_USERNAME} --noprompt"
            if (rc != 0) {
                error 'org deletion request failed'
            }
        }
    }
}

def runApexTests() {
    stage('Run Apex Test') {
        sh "mkdir -p ${RUN_ARTIFACT_DIR}"
        timeout(time: 120, unit: 'SECONDS') {
            rc = sh returnStatus: true, script: "sfdx force:apex:test:run --testlevel RunLocalTests --outputdir ${RUN_ARTIFACT_DIR} --resultformat tap --targetusername ${SFDC_USERNAME} --wait"
            if (rc != 0) {
                error 'apex test run failed'
            }
        }
    }
}

def pushSource() {
    stage('Push Source Test Org') {
        rc = sh returnStatus: true, script: "sfdx force:source:push --targetusername ${SFDC_USERNAME}"
        if (rc != 0) {
            error 'push failed'
        }
    }
}