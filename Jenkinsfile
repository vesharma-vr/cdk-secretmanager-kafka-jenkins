// Jenkins CI/CD pipeline for KafkaSimpleCdkStack (single-node KRaft Kafka on EC2).
//
//   Build & Verify    install -> unit tests -> cdk synth -> cfn-lint -> cdk diff
//   Approve           manual gate (main only)
//   Deploy & Verify   cdk deploy of the exact cdk.out reviewed above -> smoke test
//   Destroy           ACTION=destroy: typed confirmation -> cdk destroy
//
// Every branch / PR runs Build & Verify. Only `main` (or a non-multibranch job)
// can deploy or destroy. See "CI/CD with Jenkins" in the README for setup.

pipeline {
    agent none

    parameters {
        choice(name: 'ACTION', choices: ['deploy', 'plan', 'destroy'],
               description: 'deploy = build, test, approve, deploy, smoke test. ' +
                            'plan = stop after cdk diff. destroy = tear the stack down.')
        string(name: 'AWS_CREDENTIALS_ID', defaultValue: 'aws-kafka-deployer',
               description: 'Jenkins "AWS Credentials" id. Leave empty to use the agent\'s own ' +
                            'IAM role / environment credentials.')
        booleanParam(name: 'SKIP_APPROVAL', defaultValue: false,
                     description: 'Deploy without the manual approval gate.')
        booleanParam(name: 'RUN_SMOKE_TESTS', defaultValue: true,
                     description: 'After deploy, verify the broker end to end over SSM.')
        booleanParam(name: 'FORCE_DELETE_SECRET', defaultValue: false,
                     description: 'destroy only: also purge the broker secret now instead of ' +
                                  'leaving it in its 30-day recovery window (lets you redeploy immediately).')
    }

    options {
        // Two builds racing on one CloudFormation stack is never what you want.
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '10'))
        timestamps()
        timeout(time: 2, unit: 'HOURS')
    }

    environment {
        AWS_REGION         = 'us-east-1'
        AWS_DEFAULT_REGION = 'us-east-1'
        STACK_NAME         = 'KafkaSimpleCdkStack'
        SECRET_NAME        = 'kafka-simple-cdk-broker-credentials'
    }

    stages {
        stage('Build & Verify') {
            when {
                beforeAgent true
                expression { params.ACTION != 'destroy' }
            }
            agent { dockerfile { dir 'ci' } }
            stages {
                stage('Install') {
                    steps {
                        installDeps()
                    }
                }

                stage('Unit tests') {
                    steps {
                        sh '. .venv/bin/activate && python -m pytest -v --junitxml=reports/junit.xml'
                    }
                    post {
                        always { junit 'reports/junit.xml' }
                    }
                }

                stage('Synth') {
                    steps {
                        withAws {
                            // The stack looks up the default VPC at synth time. CI should be
                            // served entirely from the committed cdk.context.json; if synth had
                            // to do a fresh lookup, the deploy would not be reproducible from git.
                            sh '''
                                cp cdk.context.json reports/cdk.context.before.json
                                . .venv/bin/activate
                                cdk synth --quiet
                            '''
                        }
                        script {
                            if (sh(script: 'cmp -s cdk.context.json reports/cdk.context.before.json', returnStatus: true) != 0) {
                                sh 'cp cdk.context.json reports/cdk.context.json'
                                unstable('cdk synth changed cdk.context.json (new context lookup). Commit the ' +
                                         'archived reports/cdk.context.json so deploys are reproducible.')
                            }
                        }
                    }
                }

                stage('Static analysis') {
                    steps {
                        // Fail on errors; warnings are printed but do not break the build.
                        sh '. .venv/bin/activate && cfn-lint --non-zero-exit-code error -- cdk.out/*.template.json'
                    }
                }

                stage('Diff') {
                    steps {
                        withAws {
                            sh '''
                                status=$(aws cloudformation describe-stacks --stack-name CDKToolkit \
                                    --query 'Stacks[0].StackStatus' --output text 2>/dev/null || true)
                                case "$status" in
                                    CREATE_COMPLETE|UPDATE_COMPLETE|UPDATE_ROLLBACK_COMPLETE)
                                       echo "CDK bootstrap OK ($status)" ;;
                                    *) echo "Account/region is not CDK-bootstrapped (CDKToolkit: '${status:-missing}')."
                                       echo "Run once: cdk bootstrap aws://<ACCOUNT_ID>/${AWS_REGION}"
                                       exit 1 ;;
                                esac
                                cdk diff --app cdk.out "$STACK_NAME" > reports/cdk-diff.txt 2>&1 \
                                    || { cat reports/cdk-diff.txt; exit 1; }
                                cat reports/cdk-diff.txt
                            '''
                        }
                    }
                }
            }
            post {
                success {
                    stash name: 'cloud-assembly', includes: 'cdk.out/**'
                }
                always {
                    archiveArtifacts artifacts: 'reports/**, cdk.out/*.template.json', allowEmptyArchive: true
                }
                cleanup {
                    deleteDir()
                }
            }
        }

        stage('Approve') {
            when {
                beforeAgent true
                allOf {
                    expression { params.ACTION == 'deploy' && !params.SKIP_APPROVAL }
                    expression { isDeployBranch() }
                    expression { currentBuild.currentResult == 'SUCCESS' }
                }
            }
            // No agent: waiting for a human must not hold an executor.
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: "Deploy ${env.STACK_NAME} to ${env.AWS_REGION}? " +
                                   "Review cdk-diff.txt in this build's artifacts first.",
                          ok: 'Deploy'
                }
            }
        }

        stage('Deploy & Verify') {
            when {
                beforeAgent true
                allOf {
                    expression { params.ACTION == 'deploy' }
                    expression { isDeployBranch() }
                    // An UNSTABLE Build & Verify (e.g. uncommitted context drift) blocks deploy.
                    expression { currentBuild.currentResult == 'SUCCESS' }
                }
            }
            agent { dockerfile { dir 'ci' } }
            stages {
                stage('Deploy') {
                    steps {
                        // Deploy the stashed cloud assembly -- the exact templates that were
                        // linted and diffed above -- rather than re-synthesizing.
                        sh 'rm -rf cdk.out reports && mkdir -p reports'
                        unstash 'cloud-assembly'
                        withAws {
                            sh '''
                                cdk deploy --app cdk.out "$STACK_NAME" \
                                    --require-approval never \
                                    --outputs-file reports/cdk-outputs.json
                            '''
                        }
                    }
                }

                stage('Smoke test') {
                    when { expression { params.RUN_SMOKE_TESTS } }
                    steps {
                        withAws {
                            sh 'bash scripts/smoke_test.sh reports/cdk-outputs.json'
                        }
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
                }
                cleanup {
                    deleteDir()
                }
            }
        }

        stage('Confirm destroy') {
            when {
                beforeAgent true
                allOf {
                    expression { params.ACTION == 'destroy' }
                    expression { isDeployBranch() }
                }
            }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    script {
                        def typed = input message: "Permanently destroy ${env.STACK_NAME} in ${env.AWS_REGION}?",
                                          ok: 'Destroy',
                                          parameters: [string(name: 'CONFIRM_STACK_NAME',
                                                              description: "Type ${env.STACK_NAME} to confirm")]
                        if (typed?.trim() != env.STACK_NAME) {
                            error("Confirmation '${typed}' does not match ${env.STACK_NAME}; nothing destroyed.")
                        }
                    }
                }
            }
        }

        stage('Destroy') {
            when {
                beforeAgent true
                allOf {
                    expression { params.ACTION == 'destroy' }
                    expression { isDeployBranch() }
                }
            }
            agent { dockerfile { dir 'ci' } }
            steps {
                installDeps()
                withAws {
                    sh '''
                        . .venv/bin/activate
                        cdk destroy "$STACK_NAME" --force
                        status=$(aws cloudformation describe-stacks --stack-name "$STACK_NAME" \
                            --query 'Stacks[0].StackStatus' --output text 2>/dev/null || echo GONE)
                        echo "Stack status after destroy: $status"
                        case "$status" in GONE|DELETE_COMPLETE) ;; *) exit 1 ;; esac
                    '''
                    script {
                        if (params.FORCE_DELETE_SECRET) {
                            // See README "Secrets Manager gotcha": the fixed secret name stays
                            // reserved for 30 days unless purged.
                            sh '''
                                aws secretsmanager delete-secret --secret-id "$SECRET_NAME" \
                                    --force-delete-without-recovery \
                                || echo "Secret $SECRET_NAME not found -- nothing to purge."
                            '''
                        }
                    }
                }
            }
            post {
                cleanup {
                    deleteDir()
                }
            }
        }
    }

    post {
        success {
            echo "${params.ACTION} of ${env.STACK_NAME} succeeded."
        }
        failure {
            echo "${params.ACTION} of ${env.STACK_NAME} FAILED -- see the stage logs and archived reports/."
        }
    }
}

// Deploys and destroys are only allowed from main. BRANCH_NAME is unset in a
// plain (non-multibranch) Pipeline job, which is treated as the deploy job.
def isDeployBranch() {
    return env.BRANCH_NAME == null || env.BRANCH_NAME == 'main'
}

def installDeps() {
    sh '''
        rm -rf reports && mkdir -p reports
        python3 -m venv .venv
        . .venv/bin/activate
        pip install --quiet --upgrade pip
        pip install --quiet -r requirements-dev.txt
        cdk --version
    '''
}

// Binds the Jenkins AWS credential (CloudBees AWS Credentials plugin) as
// AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY, or falls through to whatever
// credentials the agent already has (instance profile, IRSA, env).
def withAws(Closure body) {
    def credId = params.AWS_CREDENTIALS_ID?.trim()
    if (credId) {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: credId]]) {
            body()
        }
    } else {
        body()
    }
}
