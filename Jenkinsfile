#!/usr/bin/env groovy

@Library('tenable.common')

def projectProperties = [
    [$class: 'BuildDiscarderProperty',strategy: [$class: 'LogRotator', numToKeepStr: '5']],disableConcurrentBuilds(),
    [$class: 'ParametersDefinitionProperty', parameterDefinitions: [[$class: 'StringParameterDefinition', defaultValue: 'io/qa-milestone', description: '', name: 'SITE_BRANCH']]]
]

properties(projectProperties)

import com.tenable.jenkins.Slack
import com.tenable.jenkins.common.Common
import com.tenable.jenkins.Constants
import com.tenable.jenkins.github.GitHub

Constants global = new Constants()
Common common = new Common()
Slack slack  = new Slack()
def fmt = slack.helper()
def auser = ''
GitHub github = new GitHub()
String releasebranch = "master"

// This is an extra security to avaoid pushes (even from master) until we are ready
Boolean relpush = false

def push = env.BRANCH_NAME == releasebranch && relpush ? "push" : ""

try {
    node(global.DOCKERNODE) {
        common.cleanup()

        // We check and see if the release is not yet used
        stage("prepare") {
            Boolean fail = push ? true : false

            def SCM = checkout scm

            props = readProperties(file : 'tenable_io/__init__.py')
            String ver = props['__version__']
            ver = ver.replace('"', '')

            if (github.checkRelease(ver, SCM, fail)) {
                echo "Warning: Tag ${ver} already exists!"
            }
        }
    }

    node(global.DOCKERNODE) {
        common.cleanup()

        // Pull the automation framework from develop
        stage('scm auto') {
            dir('tenableio-sdk') {
                checkout scm

                // create the pypi credentials file
                if (push) {
                    withCredentials([[$class:'UsernamePasswordMultiBinding', credentialsId:'PYPICREDS',
                                      usernameVariable:'USR', passwordVariable:'PWD']]) {
                    sh """
echo "[pypi]" > ~/.pypirc
echo "repository=https://pypi.python.org/pypi" >> ~/.pypirc
echo "username=${USR}" >> ~/.pypirc
echo "password=${PWD}" >> ~/.pypirc
"""
                    }
                }
            }
            dir('automation') {
                git(branch:'develop', changelog:false, credentialsId:global.BITBUCKETUSER, poll:false,
                    url:'ssh://git@stash.corp.tenablesecurity.com:7999/aut/automation-tenableio.git')
            }
            dir('site') {
                git(branch:params.SITE_BRANCH, changelog:false, credentialsId:global.BITBUCKETUSER, poll:false,
                    url:'ssh://git@stash.corp.tenablesecurity.com:7999/aut/site-configs.git')
            }
        }

        docker.withRegistry(global.AWS_DOCKER_REGISTRY) {
            docker.image('ci-vulnautomation-base:1.0.9').inside('-u root') {
                stage('build auto') {
                    common.prepareGit()

                    sshagent([global.BITBUCKETUSER]) {
                        timeout(time: 60, unit: 'MINUTES') {
                            try {
                                sh '''
cd automation || exit 1
export JENKINS_NODE_COOKIE=
unset JENKINS_NODE_COOKIE

python3 autosetup.py catium --all --no-venv 2>&1

export PYTHONHASHSEED=0
export PYTHONPATH=.
export CAT_USE_GRID=true

python3 tenableio/commandline/sdk_test_container.py --create_container --python --agents 5

cd ../tenableio-sdk || exit 1
pip3 install -r requirements.txt || exit 1
py.test tests --junitxml=test-results-junit.xml || exit 1

pip3 install wheel

python setup.py bdist_wheel

if [ -n "${push}" ]
then
  pip3 install twine
  twine upload dist/*
fi
'''
                            }
                            finally {
                                step([$class: 'JUnitResultArchiver', testResults: 'tenableio-sdk/*.xml'])
                            }
                        }
                    }
                }
            }
        }

        currentBuild.result = currentBuild.result ?: 'SUCCESS'
    }

    if (push) {
        node(global.DOCKERNODE) {
            common.cleanup()

            stage("tagRepo") {
                def SCM = checkout scm

                props = readProperties(file : 'tenable_io/__init__.py')
                String ver = props['__version__']
                ver = ver.replace('"', '')

                github.createRelease(ver, SCM)
            }
        }
    }
}
catch (exc) {
    if (currentBuild.result == null || currentBuild.result == 'ANORTED') {
        // Try to detect if the build was aborted
        if (common.wasAborted(this)) {
            currentBuild.result = 'ABORTED'
            auser = common.jenkinsAbortUser()

            if (auser) {
               auser = '\nAborted by ' + auser
            }
        }
        else {
            currentBuild.result = 'FAILURE'
        }
    }
    else {
        currentBuild.result = 'FAILURE'
    }
    throw exc
}
finally {
    String tests = common.jenkinsTestResults()
    String took  = '\nTook: ' + common.jenkinsDuration()

    currentBuild.result = currentBuild.result ?: 'FAILURE'

    messageAttachment = fmt.getDecoratedFinishMsg(this,
        'Tenable SDK Python build finished with result: ',
        "Built off branch ${env.BRANCH_NAME}" + tests + took + auser)

    messageAttachment.channel = '#sdk'
    slack.postMessage(this, messageAttachment)
}
