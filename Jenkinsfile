
pipeline {
    agent any

    environment {
        ECR_REPO = "131696161928.dkr.ecr.us-east-2.amazonaws.com/abode-webapp"
        ECR_REGION = "us-east-2"
        SSM_REGION = "us-east-1"
        PROD_INSTANCE = "i-05a643776b1b7397a"
    }

    stages {
        stage('build') {
            steps {
                echo 'Building Docker image...'
                sh 'docker build -t abode-webapp .'
            }
        }

        stage('test') {
            steps {
                echo 'Testing Docker container...'
                sh '''
                    docker rm -f abode-jenkins-test || true
                    docker run -d --name abode-jenkins-test -p 8082:80 abode-webapp
                    sleep 5
                    curl -f http://localhost:8082
                    docker rm -f abode-jenkins-test
                '''
            }
        }

        stage('prod') {
            when {
                expression {
                    env.BRANCH_NAME == 'master' ||
                    env.GIT_BRANCH == 'origin/master'
                }
            }
            steps {
                echo 'Pushing image to ECR...'

                sh '''
                    aws ecr get-login-password --region $ECR_REGION |
                    docker login --username AWS --password-stdin 131696161928.dkr.ecr.us-east-2.amazonaws.com

                    docker tag abode-webapp:latest $ECR_REPO:latest
                    docker push $ECR_REPO:latest
                '''

                echo 'Deploying to Production through SSM...'

                sh '''
                    COMMAND_ID=$(aws ssm send-command \
                        --region $SSM_REGION \
                        --instance-ids $PROD_INSTANCE \
                        --document-name AWS-RunShellScript \
                        --comment "Jenkins Abode deployment" \
                        --parameters '{"commands":["aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 131696161928.dkr.ecr.us-east-2.amazonaws.com","docker pull 131696161928.dkr.ecr.us-east-2.amazonaws.com/abode-webapp:latest","docker rm -f abode-production || true","docker run -d --name abode-production -p 80:80 --restart always 131696161928.dkr.ecr.us-east-2.amazonaws.com/abode-webapp:latest","docker image prune -f"]}' \
                        --query "Command.CommandId" \
                        --output text)

                    echo "SSM Command ID: $COMMAND_ID"

                    STATUS="Pending"

                    for attempt in $(seq 1 60); do
                        STATUS=$(aws ssm get-command-invocation \
                            --region $SSM_REGION \
                            --command-id "$COMMAND_ID" \
                            --instance-id $PROD_INSTANCE \
                            --query Status \
                            --output text 2>/dev/null || echo "Pending")

                        echo "Deployment status: $STATUS"

                        if [ "$STATUS" = "Success" ]; then
                            break
                        fi

                        if [ "$STATUS" = "Failed" ] || [ "$STATUS" = "Cancelled" ] || [ "$STATUS" = "TimedOut" ] || [ "$STATUS" = "Undeliverable" ]; then
                            aws ssm get-command-invocation \
                                --region $SSM_REGION \
                                --command-id "$COMMAND_ID" \
                                --instance-id $PROD_INSTANCE
                            exit 1
                        fi

                        sleep 5
                    done

                    if [ "$STATUS" != "Success" ]; then
                        echo "Production deployment timed out."
                        exit 1
                    fi

                    echo "Production deployment successful."
                '''
            }
        }
    }
}
