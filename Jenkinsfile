def currentSdkVersion = ""
def gitCommit = ""

pipeline {

    agent {
        label 'mac-mini'
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    gitCommit = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
                    echo "gitCommit: ${gitCommit}"
                    currentSdkVersion = sdkVersion()
                    echo "currentSdkVersion: ${currentSdkVersion}"
                }
            }
        }
        
        stage('Yarn Install Lib') {
            steps {
                sh "yarn install"
            }
        }

        stage('Bundle Install Lib') {
            steps {
                bash 'bundle install'
            }
        }
        
        stage('Pod Install Lib') {
            steps {
                sh "cd ios && pod install && cd .."
            }
        }

        stage('Android Native Unit Tests') {
            steps {
                sh 'cd android && ./gradlew clean && ./gradlew testDebugUnitTest && cd ..'
                junit 'android/build/test-results/testDebugUnitTest/*.xml'
            }
        }

        stage('iOS Native Unit Tests') {
            steps {
                bash 'bundle exec fastlane ios run_lib_tests'
            }
        }

        stage('Yarn Install TestApp') {
            steps {
                sh 'cd TestApp && yarn install && cd ..'
            }
        }

        stage('Pod Install TestApp') {
            steps {
                bash 'cd TestApp/ios && pod install && cd ../..'
            }
        }

        stage('Build Android TestApp') {
            steps {
                sh 'cd TestApp/android && ./gradlew clean && ./gradlew app:assembleDebug && cd ../..'
                // apk location: TestApp/android/app/build/outputs/apk/debug/app-debug.apk
                // TODO : Upload apk
            }
        }

        stage('Build iOS TestApp') {
            steps {
                bash 'bundle exec fastlane ios build_test_apps'
            }
        }

        stage('Android Integration Tests') {
            steps {
                try {
                    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'BrowserStack', usernameVariable: 'BROWSERSTACK_USER', passwordVariable: 'BROWSERSTACK_PASSWORD']]) {
                        sh 'BrowserStackLocal --key ' + "$BROWSERSTACK_PASSWORD" + '&'
                        sh 'curl -u ' + "$BROWSERSTACK_USER" + ":" + "$BROWSERSTACK_PASSWORD" + ' -X POST https://api-cloud.browserstack.com/app-automate/upload -F file=@TestApp/android/app/build/outputs/apk/debug/app-debug.apk -F "data={\"custom_id\":\"INAPP_RN_SDK_TEST_APP\"}"'
                        sh './integration-tests/gradlew -p integration-tests :test -Dbrowserstack.username=' + "$BROWSERSTACK_USER" + '" -Dbrowserstack.password="' + "$BROWSERSTACK_PASSWORD"
                    }
                } finally {
                    // Add the test results for statistics
                    junit 'integration-tests/build/test-results/test/*.xml'
                }
            }
        }

        stage('Clean Directory') {
            steps {
                script {
                    sh 'git reset --hard'
                    sh 'git clean -d -x -f'
                }
            }
        }
        
    }
}

String sdkVersion() {
    return sh(returnStdout: true, script: "sed -n -e '/\"version\"/ s/.*\\: *//p' package.json").trim().replaceAll("\"", "").replaceAll(",", "")
}

def bash(custom) {
    sh '''#!/bin/bash -l
    ''' + custom
}
