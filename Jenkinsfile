#!/usr/bin/env groovy

projectName = 'esp-pipeline' new code

node() {
    ansiColor('xterm') {
        docker.withRegistry(env.DOCKER_REGISTRY_URL, env.DOCKER_CREDENTIALS_ID) {
            stage('Setup') {
                java.jdk8()
                deleteDir()
                github.clone(projectName)
            }

            stage('Test') {
                testUnit()
                testIntegrationAndAcceptance()
            }

            stage('Release') {
                gradlew.release('master')
            }

            stage('Publish') {
                if (branch.isMaster()) {
                    gradlew.clean()
                    gradlew.exec('dockerBuild')
                    gradlew.publish('master')

                    def imageId = dockerw.imageId(projectName)
                    def imageVersion = gradlew.getVersion() - '-SNAPSHOT'
                    def imageName = 'esp/pipeline'
                    def imageTags = ['latest', imageVersion]

                    dockerw.push(imageId, imageName, imageTags)

                    archiveImageProperties(imageName, imageVersion)
                }
            }
        }
    }
}

def testUnit() {
    try {
        timeout(15) {
            gradlew.exec("testUnit")
        }
    } finally {
        publishJUnitReport('Unit', 'build/reports/testUnit')
    }
}

def testIntegrationAndAcceptance() {
    def composeId = "${env.BRANCH_NAME}b${env.BUILD_NUMBER}"
            .replaceAll('[^a-zA-Z0-9]', '')
            .toLowerCase()
    try {
        timeout(15) {
            gradlew.exec("composeTestIntegrationAndAcceptance -PcomposeId=${composeId}")
        }
    } catch (ignored) {
        try {
            gradlew.exec("composeCopyTestReports -PcomposeId=${composeId}")
            gradlew.exec("composeCopyTestResults -PcomposeId=${composeId}")
        } finally {
            gradlew.exec("composeDown -PcomposeId=${composeId}")
        }
    } finally {
        try {
            publishJUnitReport('Integration', 'build/compose-results/testIntegration')
            publishJUnitReport('Acceptance', 'build/compose-results/testIntegration')
            publishCucumberReport('build/compose-reports/cucumber')
        } finally {
            archiveArtifacts 'build/docker/compose.log'
            junit 'build/*-results/test*/*.xml'
        }
    }
}

def archiveImageProperties(name, version) {
    writeFile file: 'build/image.properties', text: """
image.name=${env.DOCKER_REGISTRY}/${name}
image.version=${version}
"""
    archiveArtifacts 'build/image.properties'
}

def publishJUnitReport(reportName, reportDir) {
    publishHTML([allowMissing         : false,
                 alwaysLinkToLastBuild: false,
                 keepAll              : true,
                 reportDir            : reportDir,
                 reportFiles          : 'index.html',
                 reportName           : reportName,
                 reportTitles         : ''
    ])
    checkErrors("${reportName} tests failed")
}

def publishCucumberReport(reportDir) {
    cucumber buildStatus: 'FAILURE',
            fileIncludePattern: '**/cucumber.json',
            jsonReportDirectory: reportDir

    checkErrors('Cucumber tests failed')
}

def checkErrors(message = '') {
    if (currentBuild.result == 'FAILURE' || currentBuild.currentResult == 'FAILURE') {
        error message
    }
}
