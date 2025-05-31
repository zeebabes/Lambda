pipeline {
    agent {
        docker {
            image 'amazonlinux:2023'
            args '-u root -v /tmp:/tmp -e PIP_NO_CACHE_DIR=1'
        }
    }
    environment {
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        TF_IN_AUTOMATION      = 'true'
    }
    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Install Tools') {
            steps {
                sh '''
                    yum update -y --skip-broken
                    yum install -y python3 python3-pip zip wget unzip
                    pip3 install --no-cache-dir pytest bandit
                '''
            }
        }

        stage('Setup Terraform') {
            steps {
                sh '''
                    TERRAFORM_VERSION="1.6.6"
                    wget -q https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip
                    unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip -d /usr/local/bin/
                    rm terraform_${TERRAFORM_VERSION}_linux_amd64.zip
                '''
            }
        }

        stage('Prepare Lambda Package') {
            steps {
                sh '''
                    cd lambda
                    pip3 install -r requirements.txt -t . 
                    zip -r ../infra/lambda.zip .
                '''
            }
        }

        stage('Terraform Init & Apply') {
            steps {
                dir('infra') {
                    sh '''
                        terraform init
                        terraform apply -auto-approve
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    def api_url = sh(
                        script: 'cd infra && terraform output -raw api_url',
                        returnStdout: true
                    ).trim()
                    echo "API Gateway Endpoint: ${api_url}"
                    sh "curl -s ${api_url}"
                }
            }
        }

        stage('Destroy Infrastructure') {
            steps {
                dir('infra') {
                    sh 'terraform destroy -auto-approve'
                }
            }
        }
    }

    post {
        failure {
            dir('infra') {
                sh 'terraform destroy -auto-approve'
            }
        }
        cleanup {
            cleanWs()
        }
    }
}
