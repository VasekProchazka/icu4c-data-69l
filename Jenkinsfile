def secrets = [[
        path: 'jenkins/web',
        secretValues: [[
            envVar: 'SYSTEM_TOOLS_DOCKER_CONFIG',
            vaultKey: 'docker-config']]
    ],[
        path: 'jenkins/web',
        secretValues: [[
            envVar: 'NPM_TOKEN',
            vaultKey: 'npm-token']]
]]

pipeline {

    agent {
        label 'node'
    }

    options {
        timeout(time: 20, unit: 'MINUTES')
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        SONAR_SCANNER_MIRROR = 'https://nexus-nexus.d1czos.ifortuna.cz/repository/web2_test_binaries/sonar-scanner-cli-npm/'
        NPM_TOKEN = 'REPLACED_IN_PUBLISH_STEP'

        DOCKER_DIR = 'docker-image-content'
        QUAY_REGISTRY = "czdcm-quay.lx.ifortuna.cz"
        PROJECT_NAMESPACE = "web2"
        OC_BUILD_APP_NAME = "fortunaweb-fe"

        versionPolicies = readJSON file: 'common/config/rush/version-policies.json'
        PACKAGE_VERSION = "${versionPolicies[0].version}"
        COMMIT_HASH = sh (script: "git rev-parse --short HEAD", returnStdout: true).trim()
        BUILD_TIME = new Date().format("yyyyMMdd'T'HHmmss", TimeZone.getTimeZone('UTC'))
        APP_VERSION = "${PACKAGE_VERSION}-${BUILD_TIME}-${COMMIT_HASH}"

        imageTag = "${APP_VERSION}"
        quayAppName = "${QUAY_REGISTRY}/${PROJECT_NAMESPACE}/${OC_BUILD_APP_NAME}:${APP_VERSION}"
    }
    stages {
        stage('Print build environment variables') {
            steps {
                sh "node -v"
                sh "npm -v"
                sh "env | sort"
            }
        }
        stage('Checkout branch') {
            when {
                not {
                    changeRequest()
                }
            }
            steps {
                checkout scm
            }
        }

        stage('Checkout and merge source with target for PR') {
            when {
                changeRequest()
            }
            steps {
                sh """
                    git config --global user.email 'jenkins@feg.eu'
                    git config --global user.name 'Jenkins'
                """
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "origin/${env.CHANGE_BRANCH}"]],
                    doGenerateSubmoduleConfigurations: false,
                    extensions: [[
                        $class: 'PreBuildMerge',
                        options: [
                            mergeRemote: 'origin',
                                // never create merge commit because we take commit hash from the last commit
                                fastForwardMode: 'FF',
                            mergeTarget: "${env.CHANGE_TARGET}"]],
                        [$class: 'CleanBeforeCheckout'],
                        [$class: 'PruneStaleBranch']],
                    submoduleCfg: [],
                    userRemoteConfigs: [[
                        credentialsId: 'bitbucket-global',
                        url: 'ssh://git@czdc-bitb-01.ux.ifortuna.cz:7999/web/fortunaweb-fe.git']]

                ])
            }
        }

        stage('NPM setup') {
            steps {
                sh """
                    npm config set proxy null
                    npm config set https_proxy null
                    export HTTP_PROXY=http://10.8.68.20:3128
                    export HTTPS_PROXY=http://10.8.68.20:3128
                    export NO_PROXY=nexus-nexus.d1czos.ifortuna.cz,0,1,2,3,4,5,6,7,8,9
                """
            }
        }

        stage('Publish npm package') {
            when {
              anyOf {
                branch 'master'
                branch 'develop'
			    branch 'release/*'
                branch 'hotfix/*'
              }
              not {
                  changeRequest()
              }
            }
            steps {
                script {
                    def secret = [[
                        path: 'jenkins/web',
                        secretValues: [[
                            envVar: 'NPM_TOKEN',
                            vaultKey: 'npm-token']]
                    ]]
                    withVault(vaultSecrets: secret) {
                        sh "npm publish"
                    }
                }
            }
        }
    }
}
