pipeline {
    agent {
        kubernetes {
            yaml rustAgent(nightly: false)
        }
    }

    parameters {
        booleanParam(name: 'RUN_FUZZ', defaultValue: false, description: 'Run fuzzing tests')
        booleanParam(name: 'PUBLISH', defaultValue: false, description: 'Publish if the commit message contains [RELEASE]')
    }

    stages {
        stage('Cargo Fmt and Clippy') {
            steps {
                container(name: 'rust') {
                    sh 'cargo fmt --all -- --check'
                    sh 'cargo clippy --all-features -- -D warnings'
                }
            }
        }

        stage('Run Tests') {
            steps {
                container(name: 'rust') {
                    sh 'cargo test --all-targets --all-features'
                }
            }
        }

        stage('Fuzzing') {
            when {
                expression { return params.RUN_FUZZ }
            }
            agent none
            steps {
                script {
                    podTemplate(yaml: rustAgent(nightly: true)) {
                        node(POD_LABEL) {
                            container(name: 'rust-nightly') {
                                sh 'cargo fuzz run fuzz_target_1 -- -max_total_time=600'
                            }
                        }
                    }
                }
            }
        }


        stage('Publish') {
            when {
                expression { return isReleaseBuild() || params.PUBLISH }
            }
            steps {
                container(name: 'rust') {
                    sh 'cargo publish'
                }
            }
        }
    }

    post {
        always {
            script {
                def fuzzingErrors = false
                def commitMessage = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()
                def releaseEnabled = params.PUBLISH || commitMessage.contains('[RELEASE]')

                echo "Build result: ${currentBuild.currentResult}"

                if(currentBuild.resultIsWorseOrEqualTo('FAILURE')) {
                    // Check if fuzzing has produced any errors, you need to define how to detect these.
                    // fuzzingErrors = fileExists('fuzz_errors.log')

                    if(releaseEnabled || fuzzingErrors) {
                        sendFailEmail(project: 'V12-async_engine')
                    }
                }
            }
        }
    }
}