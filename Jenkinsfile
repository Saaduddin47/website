pipeline {
    agent any

    environment {
        IMAGE_NAME = "abode-webapp"
        ECR_REPO = "131696161928.dkr.ecr.us-east-2.amazonaws.com/abode-webapp"
        AWS_REGION = "us-east-2"
        PROD_HOST = "54.159.96.67"
    }

    stages {
        stage('build') {
            steps {
                echo 'Building Docker image...'
                sh 'sudo docker build -t abode-webapp .'
            }
        }

        stage('test') {
            steps {
                echo 'Testing Docker container...'
                sh '''
                    sudo docker rm -f abode-jenkins-test || true
                    sudo docker run -d --name abode-jenkins-test -p 8082:80 abode-webapp
                    sleep 5
                    curl -f http://localhost:8082
                    sudo docker rm -f abode-jenkins-test
                '''
            }
        }

        stage('prod') {
            when {
                branch 'master'
            }
            steps {
                echo 'Deploying to production...'

                sh '''
                    aws ecr get-login-password --region us-east-2 | sudo docker login --username AWS --password-stdin 131696161928.dkr.ecr.us-east-2.amazonaws.com
                    sudo docker tag abode-webapp:latest $ECR_REPO:latest
                    sudo docker push $ECR_REPO:latest
                '''

                sshagent(credentials: ['abode-prod-ssh']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@$PROD_HOST '
                            aws ecr get-login-password --region us-east-2 | sudo docker login --username AWS --password-stdin 131696161928.dkr.ecr.us-east-2.amazonaws.com &&
                            sudo docker pull $ECR_REPO:latest &&
                            sudo docker rm -f abode-production || true
                            sudo docker run -d --name abode-production -p 80:80 --restart always $ECR_REPO:latest
                        '
                    '''
                }
            }
        }
    }
}
