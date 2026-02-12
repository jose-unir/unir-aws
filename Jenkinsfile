pipeline {
    agent any

    stages {

        stage('Get Code') {
            steps {
                sh '''
                    whoami
                    hostname
                    echo "${WORKSPACE}"
                '''
                git branch: 'develop', url: 'https://github.com/jose-unir/unir-aws.git'
            }
        }

        stage('Static') {
            steps {
                sh '''
                    whoami
                    hostname
                    echo "${WORKSPACE}"
                    flake8 --exit-zero --format=pylint src > flake8.out || true
                    ls -l flake8.out
                '''
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    recordIssues(
                        id: 'flake8',
                        qualityGates: [
                            [type: 'TOTAL', threshold: 8,  criticality: 'UNSTABLE'],
                            [type: 'TOTAL', threshold: 10, criticality: 'FAILURE']
                        ],
                        sourceCodeRetention: 'LAST_BUILD',
                        tools: [pyLint(pattern: 'flake8.out')],
                        enabledForFailure: true
                    )
                }
            }
        }

        stage('Security') {
            steps {
                sh '''
                    whoami
                    hostname
                    echo "${WORKSPACE}"
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install bandit
                    bandit -r src -f json -o bandit.json || true
                    ls -l bandit.json
                '''
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    recordIssues(
                        id: 'bandit',
                        qualityGates: [
                            [type: 'TOTAL', threshold: 2, criticality: 'UNSTABLE'],
                            [type: 'TOTAL', threshold: 4, criticality: 'FAILURE']
                        ],
                        sourceCodeRetention: 'LAST_BUILD',
                        tools: [pyLint(pattern: 'bandit.json')],
                        enabledForFailure: true
                    )
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    whoami
                    hostname
                    echo "${WORKSPACE}"
                    sam build
                    sam validate --region us-east-1
                    sam deploy --config-env staging --region us-east-1 --no-fail-on-empty-changeset
                '''
            }
        }

        stage('Rest') {
            steps {
                sh '''
                    export BASE_URL=$(aws cloudformation describe-stacks \
                        --stack-name todo-list-aws-staging \
                        --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' \
                        --region us-east-1 \
                        --output text)
                    echo "API_URL obtenida: $BASE_URL"
                    pytest -q test/integration/todoApiTest.py --junitxml=pytest-results.xml
                '''
                junit 'pytest-results.xml'
            }
        }

        stage('Promote') {
            steps {
                script {
                    def currentBranch = sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()
                    if (currentBranch == 'develop') {
                        withCredentials([usernamePassword(credentialsId: 'git-jose', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN')]) {
                            sh '''
                                git config user.name "${GIT_USERNAME}"
                                git config user.email "${GIT_USERNAME}@jenkins"
                                git remote set-url origin https://${GIT_USERNAME}:${GIT_TOKEN}@github.com/jose-unir/unir-aws.git
                                git fetch origin master || git fetch origin develop:master
                                git checkout master || git checkout -b master origin/master
                                git reset --hard origin/master
                                git merge origin/develop --no-ff --no-commit || true
                                git checkout --ours Jenkinsfile
                                git add Jenkinsfile
                                git commit -m "Merge develop into master keeping master Jenkinsfile" || true
                                git push origin master
                            '''
                        }
                    } else {
                        echo "Branch is not develop. Skipping Promote stage."
                    }
                }
            }
        }
    }
}
