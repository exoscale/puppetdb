@Library('jenkins-pipeline') _

node {
     cleanWs()

     try {
         def gitTag
         def gitBranch
         def onXenialBranch

         dir('src') {

             stage('SCM') {
                 checkout scm
                 gitTag = getGitTag()
                 gitBranch = params.REF
                 // Returns true if the current commit is present on a remote
                 // branch containing "xenial", even if that is not the branch that
                 // triggered the job
                 onXenialBranch = Integer.parseInt(sh(returnStdout: true, script: "git branch -r --contains | grep xenial | wc -l").trim()) > 0
                 echo gitTag
                 echo gitBranch
                 echo Boolean.toString(onXenialBranch)
             }

             updateGithubCommitStatus('PENDING', "${env.WORKSPACE}/src")

             stage('test') {
                 test()
             }

             stage('Build uberjar') {
                 build()
             }

             // When building a tag params.REF is set to the tag name
             // We therefore check if the commit is also on a xenial branch when building a tagged commit
             if (gitTag != "" && onXenialBranch || gitBranch.startsWith("xenial")) {
                 stage('Build package') {
                     gitPbuilder('xenial', false, '../build-area-xenial')
                 }
             }

             if (gitTag != "" && onXenialBranch) {
                 stage('Upload package') {
                     aptlyUpload('staging', 'xenial', 'main', '../build-area-xenial/*.deb')
                 }
             }

         }
     } catch (err) {
         currentBuild.result = 'FAILURE'
         throw err
     } finally {
         if (!currentBuild.result) {
             currentBuild.result = 'SUCCESS'
         }
         updateGithubCommitStatus(currentBuild.result, "${env.WORKSPACE}/src")
         cleanWs cleanWhenFailure: false
     }
}

// Test for now will just run the tests using the built-in embedded database
// Once I feel more comfortable with jenkins, we can create a setup that spawns both a postgres database
// and returns the tests against that
def test() {
    docker.withRegistry('https://registry.internal.exoscale.ch') {
        def clojure = docker.image('registry.internal.exoscale.ch/exoscale/clojure:bionic')
        clojure.pull()
        try {
            clojure.inside('-u root --net host -v /home/exec/.m2/repository:/root/.m2/repository -v /etc/puppet/ssl:/etc/puppet/ssl') {
                sh 'lein test'
            }
        } catch (err) {
            currentBuild.result = 'FAILURE'
            throw err
        }
    }
}

def build() {
    docker.withRegistry('https://registry.internal.exoscale.ch') {
        def clojure = docker.image('registry.internal.exoscale.ch/exoscale/clojure:bionic')
        clojure.pull()
        clojure.inside('-u root -v /home/exec/.m2/repository:/root/.m2/repository -e "USER=jenkins"') {
            // Installs the necessary dependencies. Can be put into a base image if we end up running this frequently
            sh 'apt-get update && apt-get install -y ruby-puppetlabs-spec-helper'
            // Downloads the packaging ruby gem in a distribution agnostic way (needed for the rake template step)
            sh 'rake package:bootstrap'
            // Builds the binary
            sh 'rake uberjar'
            // Creates the various distribution config files
            sh 'rake template'
            // Remove the original templates of the config files (confuses the debian package builder if they are still around)
            sh 'rm -rf ./ext/packaging'
            // Ensure all file permissions are set correctly, to avoid problems when building the debian package
            sh 'find ./ext/files -type f -not -perm /o+r -exec chmod a+r {} \\; -print'
        }
    }
}
