#!/usr/bin/env groovy

def majorVersion
def minorVersion
def patchVersion
def testImage
def prodImage

pipeline {
    agent any
    options {
        skipStagesAfterUnstable()
        timeout(time: 2, unit: 'HOURS')
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '10'))
    }
    stages {
        stage('Init') {
            steps {
                script {
                    def gitCommitHash = sh(returnStdout: true, script: 'git rev-parse HEAD').trim().take(7)
                    currentBuild.displayName = "#${BUILD_ID}-${gitCommitHash}"

                    patchVersion = sh(returnStdout: true, script: 'cat zokrates_cli/Cargo.toml | grep version | awk \'{print $3}\' | sed -e \'s/"//g\'').trim()
                    echo "ZoKrates patch version: ${patchVersion}"
                    def (major, minor, patch) = patchVersion.tokenize( '.' )
                    minorVersion = "${major}.${minor}"
                    majorVersion = major
                    echo "ZoKrates minor version: ${minorVersion}"
                    echo "ZoKrates major version: ${majorVersion}"
                }
            }
        }
        stage('Format') {
            agent {
                docker {
                  image 'rustlang/rust:nightly'
                }
            }
            steps {
                script {
                    sh "rustup component add rustfmt-preview"
                    sh "cargo fmt --all -- --check"
                }
            }
        }
        stage('Build') {
            steps {
                script {
                    ansiColor('xterm') {
                        def testDockerfile = 'dev.Dockerfile'
                        testImage = docker.build("zokrates/dev", "-f ${testDockerfile} .")
                        testImage.inside {
                            sh 'RUSTFLAGS="-D warnings" ./build.sh'
                        }
                    }
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    ansiColor('xterm') {
                        testImage.inside {
                            sh 'RUSTFLAGS="-D warnings" ./test.sh'
                        }
                    }
                }
            }
        }

        stage('Integration Test') {
            when {
                expression { env.BRANCH_NAME == 'master' || env.BRANCH_NAME == 'develop' }
            }
            steps {
                script {
                    ansiColor('xterm') {
                        testImage.inside {
                            sh 'RUSTFLAGS="-D warnings" ./full_test.sh'
                        }
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            when {
                expression { env.BRANCH_NAME == 'master' }
            }
            steps {
                script {
                    ansiColor('xterm') {
                        // prodImage = docker.build("zokrates/zokrates")
                        // docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-kyroy') {
                        //     prodImage.push(patchVersion)
                        //     prodImage.push(minorVersion)
                        //     if (majorVersion > '0') {
                        //         prodImage.push(majorVersion)
                        //     }
                        //     prodImage.push("latest")
                        // }
                    }
                }
            }
        }
    }
    post {
        always {
            // junit allowEmptyResults: true, testResults: '*test.xml'
            deleteDir()
        }
        changed {
            notifyStatusChange notificationRecipients: 'mail@kyroy.com', componentName: 'ZoKrates'
        }
    }
}
