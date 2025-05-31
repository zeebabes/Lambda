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
        stage('Destroy Infrastructure') {
            steps {
                dir('infra') {
                    sh '''
                        yum update -y --skip-broken
                        yum install -y wget unzip

                        TERRAFORM_VERSION="1.6.6"
                        wget -q https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip
                        unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip -d /usr/local/bin/
                        rm terraform_${TERRAFORM_VERSION}_linux_amd64.zip

                        terraform init
                        terraform destroy -auto-approve
                    '''
                }
            }
        }
    }
}
