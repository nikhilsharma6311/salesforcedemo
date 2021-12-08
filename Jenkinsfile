#!groovy



node {



def SF_CONSUMER_KEY=env.CONSUMER_KEY
def SF_USERNAME=env.SF_USERNAME
def SERVER_KEY_CREDENTIALS_ID=env.SERVER_KEY_CREDENTIALS_ID
def SF_INSTANCE_URL = env.SF_INSTANCE_URL ?: "https://test.salesforce.com"
def DELTACHANGES = 'deltachanges'
def DEPLOYDIR = 'toDeploy'
def APIVERSION = '52.0'
def toolbelt = tool 'sfdxtool'

//----------------------------------------------------------------------
//Check if Previous and Latest commit IDs are provided.
//----------------------------------------------------------------------

if ((params.PreviousCommitId == '') || (params.LatestCommitId == ''))
{
catchError(buildResult: 'FAILURE', stageResult: 'FAILURE')
{
println('Please enter both Previous and Latest commit IDs')
}
//error("Please enter both Previous and Latest commit IDs")
}




//----------------------------------------------------------------------
//Check if Previous and Latest commit IDs are same.
//----------------------------------------------------------------------

if (params.PreviousCommitId == params.LatestCommitId)
{
catchError(buildResult: 'FAILURE', stageResult: 'FAILURE')
{
println('Please enter both Previous and Latest commit IDs')
}
//error("Previous and Latest Commit IDs can't be same.")
}




//----------------------------------------------------------------------
//Check if Test classes are mentioned in case of RunSpecifiedTests.
//----------------------------------------------------------------------

if (TestLevel=='RunSpecifiedTests')
{
if (params.SpecifyTestClass == '')
{
catchError(buildResult: 'FAILURE', stageResult: 'FAILURE')
{
println('Please enter both Previous and Latest commit IDs')
}
//error("Please Specify Test classes")
}
}



stage('Clean Workspace') {
try {
deleteDir()
}
catch (Exception e) {
println('Unable to Clean WorkSpace.')
}
}
// -------------------------------------------------------------------------
// Check out code from source control.
// -------------------------------------------------------------------------



stage('checkout source') {
checkout scm
}




// -------------------------------------------------------------------------
// Run all the enclosed stages with access to the Salesforce
// JWT key credentials.
// -------------------------------------------------------------------------



withEnv(["HOME=${env.WORKSPACE}"]) {

withCredentials([file(credentialsId: SERVER_KEY_CREDENTIALS_ID, variable: 'server_key_file')]) {
// -------------------------------------------------------------------------
// Authenticate to Salesforce using the server key.
// -------------------------------------------------------------------------



stage('Authorize to Salesforce') {
rc = command "${toolbelt}/sfdx force:auth:logout --targetusername ${SF_USERNAME} -p"
rc = command "${toolbelt}/sfdx force:auth:jwt:grant --instanceurl ${SF_INSTANCE_URL} --clientid ${SF_CONSUMER_KEY} --jwtkeyfile ${server_key_file} --username ${SF_USERNAME} --setalias ${SF_USERNAME}"
if (rc != 0) {
error 'Salesforce org authorization failed.'
}
}




stage('Delta changes')
{
script
{
bat "echo y | sfdx plugins:install sfpowerkit"
rc = command "${toolbelt}/sfdx sfpowerkit:project:diff --revisionfrom %PreviousCommitId% --revisionto %LatestCommitId% --output ${DELTACHANGES} --apiversion ${APIVERSION} -x"

def folder = fileExists 'DeltaChanges/force-app'
def file = fileExists 'DeltaChanges/destructiveChanges.xml'

if( folder && !file )
{
dir("${WORKSPACE}/${DELTACHANGES}")
{
println "Force-app folder exist, destructiveChanges.xml doesn't exist"
rc = command "${toolbelt}/sfdx force:source:convert -d ../${DEPLOYDIR}"
}
}
else if ( !folder && file )
{
bat "copy manifest\\package.xml ${DELTACHANGES}"
println "Force-app folder doesn't exist, destructiveChanges.xml exist"
}
else if ( folder && file )
{
dir("${WORKSPACE}/${DELTACHANGES}")
{
println "Force-app folder exist, destructiveChanges.xml exist"
if (DeploymentType=='Deploy Only')
{
println "You selected deploy only so deleting destructivechanges.xml to avoid component deletion."
bat "del /f destructiveChanges.xml"
rc = command "${toolbelt}/sfdx force:source:convert -d ../${DEPLOYDIR}"
}
else if (DeploymentType=='Delete and Deploy')
{
println "Both deletion and deployment will be performed."
rc = command "${toolbelt}/sfdx force:source:convert -d ../${DEPLOYDIR}"
bat "copy destructiveChanges.xml ..\\${DEPLOYDIR}"
}
else if (DeploymentType=='Delete Only')
{
println "You selected Delete only but force-app folder also exist. So deleting the force-app folder to avoid deployment."
bat "echo y | rmdir /s force-app"
bat "copy ..\\manifest\\package.xml ."
}
else if (DeploymentType=='Validate Only')
{
println "You selected Validate Only, so only validation will be performed."
rc = command "${toolbelt}/sfdx force:source:convert -d ../${DEPLOYDIR}"
}
}
}
else
{
println "There is nothing to be deployed or deleted."
}


}
}



stage('Validate Only')
{
if (DeploymentType=='Validate Only')
{
script
{

if (TestLevel=='NoTestRun')
{
println TestLevel
rc = command "${toolbelt}/sfdx force:mdapi:deploy -d ${DEPLOYDIR} --checkonly --wait 10 --targetusername ${SF_USERNAME} "
}
else if (TestLevel=='RunLocalTests')
{
println TestLevel
rc = command "${toolbelt}/sfdx force:mdapi:deploy -d ${DEPLOYDIR} --checkonly --wait 10 --targetusername ${SF_USERNAME} --TestLevel ${TestLevel} --verbose --loglevel fatal"
}
else if (TestLevel=='RunSpecifiedTests')
{
println TestLevel
def Testclass = SpecifyTestClass.replaceAll('\\s','')
println Testclass
rc = command "${toolbelt}/sfdx force:mdapi:deploy -d ${DEPLOYDIR} --checkonly --wait 10 --targetusername ${SF_USERNAME} --TestLevel ${TestLevel} -r ${Testclass} --verbose --loglevel fatal"
}

else (rc != 0)
{
error 'Validation failed.'
}
}
}
}



// -------------------------------------------------------------------------
// Deploy metadata and execute unit tests.
// -------------------------------------------------------------------------



stage('Deploy and Run Tests')
{
if (DeploymentType=='Deploy Only')
{
script
{
if (TestLevel=='NoTestRun')
{
println TestLevel
rc = command "${toolbelt}/sfdx force:mdapi:deploy -d ${DEPLOYDIR} --wait 10 --targetusername ${SF_USERNAME} "
}
else if (TestLevel=='RunLocalTests')
{
println TestLevel
rc = command "${toolbelt}/sfdx force:mdapi:deploy -d ${DEPLOYDIR} --wait 10 --targetusername ${SF_USERNAME} --TestLevel ${TestLevel} --verbose --loglevel fatal"
}
else if (TestLevel=='RunSpecifiedTests')
{
println TestLevel
def Testclass = SpecifiedTestsRun.replaceAll('\\s','')
println Testclass
rc = command "${toolbelt}/sfdx force:mdapi:deploy -d ${DEPLOYDIR} --wait 10 --targetusername ${SF_USERNAME} --TestLevel ${TestLevel} -r ${Testclass} --verbose --loglevel fatal"
}
else (rc != 0)
{
error 'Salesforce deployment failed.'
}
}
}
}



stage('Delete Components')
{
if (DeploymentType=='Delete Only')
{
rc = command "${toolbelt}/sfdx force:mdapi:deploy -u ${SF_USERNAME} -d ${DELTACHANGES} -w 1"

}
}



stage('Delete and Deploy')
{
if (DeploymentType=='Delete and Deploy')
{
script
{
if (TestLevel=='NoTestRun')
{
println TestLevel
rc = command "${toolbelt}/sfdx force:mdapi:deploy -d ${DEPLOYDIR} --wait 10 --targetusername ${SF_USERNAME} "
}
else if (TestLevel=='RunLocalTests')
{
println TestLevel
rc = command "${toolbelt}/sfdx force:mdapi:deploy -d ${DEPLOYDIR} --wait 10 --targetusername ${SF_USERNAME} --TestLevel ${TestLevel} --verbose --loglevel fatal"
}
else if (TestLevel=='RunSpecifiedTests')
{
println TestLevel
def Testclass = SpecifyTestClass.replaceAll('\\s','')
println Testclass
rc = command "${toolbelt}/sfdx force:mdapi:deploy -d ${DEPLOYDIR} --wait 10 --targetusername ${SF_USERNAME} --TestLevel ${TestLevel} -r ${Testclass} --verbose --loglevel fatal"
}
if (rc != 0)
{
error 'Salesforce deployment failed.'
}
}

}
}

/* stage('Save Artifacts')
{
if (DeploymentType=='Delete Only')
{

bat "xcopy ${DELTACHANGES} D:\\Artifacts /EHCI"
}
else
{
bat "xcopy ${DEPLOYDIR} D:\\Artifacts /EHCI"
}
}
*/

stage('EMail Notification')
{
bat 'chdir'
if (DeploymentType=='Delete Only')
{
dir("${WORKSPACE}/${DELTACHANGES}")
{
emailext attachmentsPattern: 'destructiveChanges.xml', attachLog: true, body: '$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS: $BUILD_URL', subject: '$DEFAULT_SUBJECT', to: '$DEFAULT_RECIPIENTS, knkumarnikhil93@gmail.com'
}
}
else if(DeploymentType=='Delete and Deploy')
{
dir("${WORKSPACE}/${DEPLOYDIR}")
{
bat 'chdir'
emailext attachmentsPattern: 'package.xml, destructiveChanges.xml', attachLog: true, body: '$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS: $BUILD_URL', subject: '$DEFAULT_SUBJECT', to: '$DEFAULT_RECIPIENTS, knkumarnikhil93@gmail.com'
}
}
else
{
dir("${WORKSPACE}/${DEPLOYDIR}")
{
bat 'chdir'
emailext attachmentsPattern: 'package.xml', attachLog: true, body: '$DEFAULT_CONTENT', subject: '$DEFAULT_SUBJECT', to: '$DEFAULT_RECIPIENTS'
}
}
}



}

}

}

def command(script) {
if (isUnix()) {
return sh(returnStatus: true, script: script);
} else {
return bat(returnStatus: true, script: script);
}
}
