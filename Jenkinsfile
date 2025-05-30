pipeline {
    agent {
        docker {
            image 'amazonlinux:2023'
            args '-u root -v /tmp:/tmp -e PIP_NO_CACHE_DIR=1'
        }
    }
    environment {
        AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        TF_IN_AUTOMATION      = 'true'
    }
    stages {
        stage('Checkout Code') {
            steps { checkout scm }
        }

        stage('Install Tools') {
            steps {
                sh '''
                    yum update -y --skip-broken
                    yum install -y python3 python3-pip zip wget unzip
                    pip3 install pytest bandit
                    rm -rf /var/cache/yum
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

        stage('Prepare Lambda') {
            steps {
                sh '''
                    cd lambda
                    pip3 install -r requirements.txt -t .
                    zip -r ../infra/lambda.zip .
                '''
            }
        }

        stage('Test Suite') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh '''
                            pip3 install -r tests/requirements.txt
                            pytest tests/unit --verbose --junitxml=unit-tests.xml
                        '''
                    }
                    post {
                        always {
                            junit 'unit-tests.xml'
                        }
                    }
                }
                stage('Security Scan') {
                    steps { 
                        sh 'bandit -r lambda' 
                    }
                }
            }
        }

        stage('Terraform Init') {
            steps { 
                dir('infra') { 
                    sh 'terraform init' 
                } 
            }
        }

        stage('Terraform Apply') {
            steps {
                dir('infra') {
                    sh 'terraform apply -auto-approve'
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
                    echo "API Endpoint: ${api_url}"
                    sh "curl -s ${api_url}"
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
        success {
            emailext(
                subject: "✅ Lambda Deployment Successful: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "The Lambda function and infrastructure were deployed successfully.",
                to: "thandonoe.ndlovu@gmail.com"
            )
        }
    }
}
