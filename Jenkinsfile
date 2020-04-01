@Library('jenkins-pipeline') _

node {
     cleanWs()

     try {
         def image
         def gitTag

         dir('src') {

             stage('SCM') {
                 checkout scm
                 gitTag = getGitTag()
             }

             updateGithubCommitStatus('PENDING', "${env.WORKSPACE}/src")

             //stage('test') {
             //    test()
             //}

             stage('Build uberjar') {
                 build()
             }

             stage('Build package') {
                 gitPbuilder('xenial', false, '../build-area-xenial')
             }

             stage('Upload package') {
                 aptlyUpload('xenial', 'main', '../build-area-xenial/*.deb')
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
        def clojure = docker.image('registry.internal.exoscale.ch/exoscale/clojure:bionic') // Is this the image we want to use?
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
            sh 'apt-get update && apt-get install -y ruby-puppetlabs-spec-helper'
            sh 'rake package:implode'
            sh 'rake package:bootstrap'
            sh 'rake template'
            sh 'rm -rf ./ext/packaging'
            sh 'find ./ext/files -type f -not -perm /o+r -exec chmod a+r {} \\; -print'
        }
    }
}
