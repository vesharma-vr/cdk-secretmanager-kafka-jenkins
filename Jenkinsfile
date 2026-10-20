// Jenkins CI for KafkaSimpleCdkStack (single-node KRaft Kafka on EC2).
//
//   Install -> Unit tests
//
// Every branch / PR runs the same stages. The pipeline needs no AWS
// credentials: the tests synthesize offline from the committed
// cdk.context.json. Deploys and tear-downs are done by hand (see the README).

pipeline {
    agent { dockerfile { dir 'ci' } }

    options {
        buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '10'))
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {
        stage('Install') {
            steps {
                sh '''
                    rm -rf reports && mkdir -p reports
                    python3 -m venv .venv
                    . .venv/bin/activate
                    pip install --quiet --upgrade pip
                    pip install --quiet -r requirements-dev.txt
                '''
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
