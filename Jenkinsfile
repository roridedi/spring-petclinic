#!/usr/bin/groovy

/**
    this section of the pipeline executes on the master, which has a lot of useful variables that we can leverage to configure our pipeline
**/
node (''){
    env.BUILD="labs-ci-cd"
    env.DEV_PROJECT = "labs-dev"
    env.PREPROD_PROJECT = "labs-test"
    env.SOURCE_CONTEXT_DIR = ""
    env.UBER_JAR_CONTEXT_DIR = "target/"
    env.MVN_COMMAND = "clean install"
    env.MVN_SNAPSHOT_DEPLOYMENT_REPOSITORY = "nexus::default::http://nexus:8081/repository/maven-snapshots"
    env.MVN_RELEASE_DEPLOYMENT_REPOSITORY = "nexus::default::http://nexus:8081/repository/maven-releases"
    env.APP_NAME = "${env.JOB_NAME}".replaceAll(/-?${env.PROJECT_NAME}-?/, '').replaceAll(/-?pipeline-?/, '').replaceAll('/','')
    env.OCP_API_SERVER = "${env.OPENSHIFT_API_URL}"
    env.OCP_TOKEN = readFile('/var/run/secrets/kubernetes.io/serviceaccount/token').trim()

}

node('jenkins-slave-mvn') {

  stage('SCM Checkout') {
    checkout scm
  }

  stage('Build App') {
	sh "mvn ${env.MVN_COMMAND} "
  }
	// assumes uber jar is created
  stage('Build Image') {
	sh "oc start-build ${env.APP_NAME} --from-dir=${env.UBER_JAR_CONTEXT_DIR} --follow"
  }

  stage('Maven Bump Patch Version') {
    def origVer = readMavenPom file: 'pom.xml'
    println "Current POM Version is: ${origVer.version}"
    sh "mvn clean build-helper:parse-version versions:set -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion}"

    def releaseVer = readMavenPom file: 'pom.xml'
    println "Dev Release POM Version is: ${releaseVer.version}"
  }
  
  stage('Git Tag') {
    env.GIT_REPO = 'master'
    gitTagPush()
  }
  stage ('Deploy to DEV') {

    openshiftTag (apiURL: "${env.OCP_API_SERVER}", authToken: "${env.OCP_TOKEN}", destStream: "${env.APP_NAME}", destTag: 'latest', destinationAuthToken: "${env.OCP_TOKEN}", destinationNamespace: "${env.DEV_PROJECT}", namespace: "${env.BUILD}", srcStream: "${env.APP_NAME}", srcTag: 'latest')

    openshiftVerifyDeployment (apiURL: "${env.OCP_API_SERVER}", authToken: "${env.OCP_TOKEN}", depCfg: "${env.APP_NAME}", namespace: "${env.DEV_PROJECT}", verifyReplicaCount: true)


}

  stage ('Deploy to PreProd') {
    input "Promote Application to PreProd?"

    openshiftTag (apiURL: "${env.OCP_API_SERVER}", authToken: "${env.OCP_TOKEN}", destStream: "${env.APP_NAME}", destTag: 'latest', destinationAuthToken: "${env.OCP_TOKEN}", destinationNamespace: "${env.PREPROD_PROJECT}", namespace: "${env.DEV_PROJECT}", srcStream: "${env.APP_NAME}", srcTag: 'latest')

    openshiftVerifyDeployment (apiURL: "${env.OCP_API_SERVER}", authToken: "${env.OCP_TOKEN}", depCfg: "${env.APP_NAME}", namespace: "${env.PREPROD_PROJECT}", verifyReplicaCount: true)

 
}
}

def gitTagPush() {
  //Make Environemnt Variable with new POM Version
  def releaseVer = readMavenPom file: 'pom.xml'
  env.POM_VERS = releaseVer.version
}
def mavenBumpMinorVersion() {
  def origVer = readMavenPom file: 'pom.xml'
  println "Current POM Version is: ${origVer.version}"
  sh "mvn clean build-helper:parse-version versions:set -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.nextMinorVersion}.0"

  def releaseVer = readMavenPom file: 'pom.xml'
  println "PreProd Release POM Version is: ${releaseVer.version}"

}
