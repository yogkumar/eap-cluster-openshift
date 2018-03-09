String getVersion() {
    def lastTag = sh(returnStdout: true, script: 'git describe --abbrev=0 --tags')
    def commitsSinceTag = sh(returnStdout: true, script: "git rev-list ${lastTag.trim()}.. --count")
    def commitId = sh(returnStdout: true, script: "git rev-parse --short HEAD")
    def buildTimestamp = new Date().format("yyyyMMddHHmmss")

    return "${lastTag.trim()}.${commitsSinceTag.trim()}-${buildTimestamp}-${commitId.trim()}"
}

try {
    timeout(time: 20, unit: 'MINUTES') {
        node('maven') {

            def releaseVersion = "1.0.${env.BUILD_NUMBER}"
            def applicationName = "eap-sampleapp"

            stage('Build') {
                dir('scm') {
                    checkout scm
                    releaseVersion = getVersion()

                    sh("./mvnw -B org.codehaus.mojo:versions-maven-plugin:2.2:set -U -DnewVersion=${releaseVersion}")
                    sh('./mvnw -B package fabric8:build -Popenshift')
                }
            }

            stage('Integration Test - deploy configuration') {
                dir('config') {
                    git(
                            url: 'https://github.com/nbyl/container-configurator.git',
                            branch: 'master'
                    )
                    sh("oc delete secret ${applicationName}-stage-config --ignore-not-found=true")
                    sh("oc create secret generic ${applicationName}-stage-config --from-file=./configuration/environment.properties,./configuration/app/standalone/configuration/sso/sso.keystore")
                }
            }

            stage('Integration Test - deploy application') {
                dir('scm') {
                    sh("oc process -f src/main/openshift/application-template.yaml -p APPLICATION_NAME=${applicationName}-stage -p IMAGE_VERSION=${releaseVersion}| oc apply -f -")
                    openshiftDeploy(depCfg: "${applicationName}-stage")
                }
            }

            stage('Integration Test - deploy selenium') {
                dir('scm') {
                    sh("oc process -f src/main/openshift/selenium-standalone-chrome.yaml | oc apply -f -")
                }
            }

            stage('Integration Test - run tests') {
                dir('scm') {
                    sh("./mvnw -B org.apache.maven.plugins:maven-failsafe-plugin:integration-test org.apache.maven.plugins:maven-failsafe-plugin:verify -P acceptance-test -DacceptanceTest.hubUrl=http://selenium-standalone-chrome:4444/wd/hub -DacceptanceTest.baseUrl=http://eap-sampleapp-stage/haexample")
                    archiveArtifacts artifacts: 'target/failsafe-reports/*.*', fingerprint: true
                    junitJUnitResultArchiver testResults: 'target/failsafe-reports/*.xml'
                }

            }

            stage('Integration Test - undeploy selenium') {
                dir('scm') {
                    sh("oc process -f src/main/openshift/selenium-standalone-chrome.yaml | oc delete -f -")
                }
            }

            stage('Integration Test - teardown stage') {
                dir('scm') {
                    sh("oc process -f src/main/openshift/application-template.yaml -p APPLICATION_NAME=${applicationName}-stage -p IMAGE_VERSION=${releaseVersion}| oc delete -f -")
                }
            }

        }
    }
} catch (err) {
    echo "in catch block"
    echo "Caught: ${err}"
    currentBuild.result = 'FAILURE'
    throw err
}
