pipeline {

    agent any

    environment {
        // Azure Storage
        STORAGE_ACCOUNT = "waxjenkinsstorage123"
        FILE_SHARE = "jenkins-share"

        // Staging (ACI)
        ACI_URL = "http://wax-nginx-demo-2026.centralindia.azurecontainer.io/"

        // Production EC2 Private IPs
        WAX_SERVER_1 = "10.0.3.39"
        WAX_SERVER_2 = "10.0.4.206"

        REMOTE_DIR = "/var/www/html"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Deploy to Azure Staging') {
            steps {
                withCredentials([
                    string(credentialsId: 'azure-private-key', variable: 'STORAGE_KEY')
                ]) {
                    sh '''
                        echo "Uploading files to Azure File Share..."

                        az storage file upload-batch \
                            --account-name $STORAGE_ACCOUNT \
                            --account-key $STORAGE_KEY \
                            --destination $FILE_SHARE \
                            --source .
                    '''
                }
            }
        }

        stage('Wait for ACI') {
            steps {
                echo "Waiting for Azure File Share changes to be reflected..."
                sleep(time: 20, unit: 'SECONDS')
            }
        }

        stage('HTTP Availability Test') {
            steps {
                sh '''
                    STATUS=$(curl -o /dev/null -s -w "%{http_code}" $ACI_URL)

                    if [ "$STATUS" != "200" ]; then
                        echo "ERROR: ACI returned HTTP $STATUS"
                        exit 1
                    fi

                    echo "ACI is reachable."
                '''
            }
        }

        stage('Content Assertion') {
            steps {
                sh '''
                    PAGE=$(curl -s $ACI_URL)

                    echo "$PAGE"

                    # Change "Welcome" to text that exists on your homepage
                    echo "$PAGE" | grep -q "Welcome"

                    echo "Content validation passed."
                '''
            }
        }

        stage('Deploy to Production') {
            steps {
                sshagent(['wax-private-key']) {
                    sh '''
                        echo "Deploying to Web Server 1..."

                        rsync -avz --delete \
                            --rsync-path="sudo rsync" \
                            --exclude='.git' \
                            --exclude='Jenkinsfile' \
                            -e "ssh -o StrictHostKeyChecking=no" \
                            ./ ubuntu@$WAX_SERVER_1:$REMOTE_DIR/

                        ssh -o StrictHostKeyChecking=no ubuntu@$WAX_SERVER_1 "
                            sudo chown -R www-data:www-data $REMOTE_DIR &&
                            sudo systemctl restart apache2
                        "

                        echo "Deploying to Web Server 2..."

                        rsync -avz --delete \
                            --rsync-path="sudo rsync" \
                            --exclude='.git' \
                            --exclude='Jenkinsfile' \
                            -e "ssh -o StrictHostKeyChecking=no" \
                            ./ ubuntu@$WAX_SERVER_2:$REMOTE_DIR/

                        ssh -o StrictHostKeyChecking=no ubuntu@$WAX_SERVER_2 "
                            sudo chown -R www-data:www-data $REMOTE_DIR &&
                            sudo systemctl restart apache2
                        "
                    '''
                }
            }
        }
    }

    post {

        success {
            echo "Pipeline completed successfully."
        }

        failure {
            echo "Pipeline failed. Production deployment was skipped."
        }

        always {
            cleanWs()
        }
    }
}
