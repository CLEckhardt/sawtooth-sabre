#!groovy

// Copyright 2017 Intel Corporation
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.
// ------------------------------------------------------------------------------

pipeline {
    agent {
        node {
            label '0-7'
            customWorkspace "workspace/${env.BUILD_TAG}"
        }
    }

    triggers {
        cron(env.BRANCH_NAME == '0-7' ? 'H 3 * * *' : '')
    }

    options {
        timestamps()
        buildDiscarder(logRotator(daysToKeepStr: '31'))
    }

    environment {
        ISOLATION_ID = sh(returnStdout: true, script: 'printf $BUILD_TAG | sha256sum | cut -c1-64').trim()
        COMPOSE_PROJECT_NAME = sh(returnStdout: true, script: 'printf $BUILD_TAG | sha256sum | cut -c1-64').trim()
    }

    stages {
        stage('Check User Authorization') {
            steps {
                readTrusted 'bin/authorize-cicd'
                sh './bin/authorize-cicd "$CHANGE_AUTHOR" /etc/jenkins-authorized-builders'
            }
            when {
                not {
                    branch '0-7'
                }
            }
        }

        stage('Check for Signed-Off Commits') {
            steps {
                sh '''#!/bin/bash -l
                    if [ -v CHANGE_URL ] ;
                    then
                        temp_url="$(echo $CHANGE_URL |sed s#github.com/#api.github.com/repos/#)/commits"
                        pull_url="$(echo $temp_url |sed s#pull#pulls#)"

                        IFS=$'\n'
                        for m in $(curl -s "$pull_url" | grep "message") ; do
                            if echo "$m" | grep -qi signed-off-by:
                            then
                              continue
                            else
                              echo "FAIL: Missing Signed-Off Field"
                              echo "$m"
                              exit 1
                            fi
                        done
                        unset IFS;
                    fi
                '''
            }
        }

        stage('Run Lint') {
            steps {
              sh 'docker build . -f docker/lint -t lint-sabre:$ISOLATION_ID'
              sh 'docker run --rm -v $(pwd):/project/sawtooth-sabre lint-sabre:$ISOLATION_ID'
            }
        }

        stage('Build Sabre') {
            steps {
                sh 'docker-compose -f docker-compose-installed.yaml build sabre-cli'
                sh 'docker-compose -f docker-compose-installed.yaml build sabre-tp'
                sh 'VERSION=AUTO_STRICT REPO_VERSION=$(bin/get_version) docker-compose -f docker-compose-installed.yaml build intkey_multiply'

            }
        }

        stage('Test Sabre') {
            steps {
                sh 'docker-compose -f docker/unit-test.yaml up --build --exit-code-from unit-test-sabre'
                sh 'docker-compose -f docker-compose.yaml -f integration/sabre_test.yaml up --build --abort-on-container-exit --exit-code-from test_sabre'
            }
        }

        stage('Create Git Archive') {
            steps {
                sh '''
                    REPO=$(git remote show -n origin | grep Fetch | awk -F'[/.]' '{print $6}')
                    VERSION=`git describe --dirty`
                    git archive HEAD --format=zip -9 --output=$REPO-$VERSION.zip
                    git archive HEAD --format=tgz -9 --output=$REPO-$VERSION.tgz
                '''
            }
        }

        stage ('Build Documentation') {
            steps {
                sh 'docker-compose -f docs/docker-compose.yaml up'
                sh 'docker-compose -f docs/docker-compose.yaml down'
            }
        }

        stage('Build Archive Artifacts') {
            steps {
                sh 'mkdir -p build/debs'
                sh 'docker run --rm -v $(pwd)/build/debs:/build/debs --entrypoint "/bin/bash" sawtooth-sabre-cli:${ISOLATION_ID} "-c" "cp /tmp/*.deb /build/debs"'
                sh 'docker run --rm -v $(pwd)/build/debs:/build/debs --entrypoint "/bin/bash" sawtooth-sabre-tp:${ISOLATION_ID} "-c" "cp /tmp/*.deb /build/debs"'
                sh 'docker run --rm -v $(pwd)/build/scar:/build/scar --entrypoint "/bin/bash" intkeym-scar:${ISOLATION_ID} "-c" "cp /tmp/*.scar /build/scar"'
            }
        }
    }
    post {
        always {
            sh 'docker-compose -f docker/unit-test.yaml down'
            sh 'docker-compose -f docker-compose.yaml -f integration/sabre_test.yaml down'
            sh 'docker-compose -f docs/docker-compose.yaml down'
        }
        success {
            archiveArtifacts artifacts: '*.tgz, *.zip, build/debs/*.deb, build/scar/*.scar, docs/build/html/**'
        }
        aborted {
            error "Aborted, exiting now"
        }
        failure {
            error "Failed, exiting now"
        }
    }
}
