pipeline {
    agent any

    stages {
        stage('Get Code') {
            steps {
                sh '''
                whoami
                hostname
                echo "${WORKSPACE}}"
                '''
                git branch: 'master', url: 'https://github.com/jose-unir/unir-aws.git'
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
            sam deploy --config-env production --region us-east-1 --no-fail-on-empty-changeset
            '''
            }
        }

        stage('Rest') {
            steps {
                sh '''
                export BASE_URL=$(aws cloudformation describe-stacks \
                --stack-name todo-list-aws-production \
                --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' \
                --region us-east-1 \
                --output text)
                echo "API_URL obtenida: $BASE_URL"
                pytest test/integration/todoApiTest.py -m read_only -q --tb=short --disable-warnings --maxfail=1 --junitxml=pytest-results.xml
                '''
                junit 'pytest-results.xml'
            }
        }
    }
}
