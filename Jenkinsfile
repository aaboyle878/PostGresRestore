pipeline {
    agent any
    parameters {
        string(name: 'BACKUP_FILE', defaultValue: 'postgres_backup_2024', description: 'Name of the backup file to restore')
    }
    environment {
        RESTORE_DIR = "/tmp/postgres_restore"
        DATA_DIR = "/opt/cardano/cnode/guild-db/pgdb/"
        SLACK_CHANNEL = '#jenkins-notifications'
    }
    stages {
        stage('Retrieve Secrets') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_REGION', variable: 'AWS_REGION'),
                    string(credentialsId: 'EC2_REGION', variable: 'EC2_REGION'),
                    string(credentialsId: 'S3_BUCKET', variable: 'S3_BUCKET'),
                    string(credentialsId: 'EC2_HOST', variable: 'EC2_HOST'),
                    string(credentialsId: 'INSTANCE_ID', variable: 'INSTANCE_ID'),
                    string(credentialsId: 'NETWORK', variable: 'NETWORK'),
                    string(credentialsId: 'SLACK', variable: 'SLACK'),
                    string(credentialsId: 'IAM_ROLE', variable: 'IAM_ROLE')
                ]) {
                    script {
                        // Set environment variables so they are available throughout the pipeline
                        env.AWS_REGION = "${AWS_REGION}"
                        env.EC2_REGION = "${EC2_REGION}"
                        env.S3_BUCKET = "${S3_BUCKET}"
                        env.EC2_HOST = "${EC2_HOST}"
                        env.INSTANCE_ID = "${INSTANCE_ID}"
                        env.NETWORK = "${NETWORK}"
                        env.SLACK = "${SLACK}"
                        env.IAM_ROLE = "${IAM_ROLE}"
                    }
                }
            }
        }
        stage('Check and Clean Directories') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    sh """
                    ssh ubuntu@${EC2_HOST} << 'EOF'
                    set -e
                    echo 'Checking and cleaning directories...'

                    # Ensure restore directory exists
                    sudo mkdir -p ${RESTORE_DIR}
                    sudo chmod -R 755 ${RESTORE_DIR}

                    # Clean restore directory if it has files
                    if [ -d ${RESTORE_DIR} ] && [ "\$(ls -A ${RESTORE_DIR})" ]; then
                        sudo rm -rf ${RESTORE_DIR}/*
                        echo 'Restore directory cleaned.'
                    fi

                    # Ensure data directory exists and clean it
                    sudo mkdir -p ${DATA_DIR}
                    sudo chmod -R 755 ${DATA_DIR}
                    if [ -d ${DATA_DIR} ] && [ "\$(ls -A ${DATA_DIR})" ]; then
                        sudo rm -rf ${DATA_DIR}/*
                        echo 'Data directory cleaned.'
                    fi

                    EOF
                    """
                }
            }
        }
        stage('Download Backup from S3') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    retry(3) {
                        sh """
                        ssh ubuntu@${EC2_HOST} <<'EOF'
                        set -e
                        echo 'Downloading backup tarball...'
                        aws s3 cp s3://${S3_BUCKET}/${NETWORK}/${BACKUP_FILE}.tar.gz /tmp/
                        echo 'Backup tarball downloaded.'
                        EOF
                        """
                    }
                }
            }
        }
        stage('Extract Backup Tarball') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    retry(2) {
                        sh """
                        ssh ubuntu@${EC2_HOST} <<'EOF'
                        set -e
                        echo 'Extracting backup tarball...'
                        mkdir -p ${RESTORE_DIR}
                        tar -xzf /tmp/${BACKUP_FILE}.tar.gz -C ${RESTORE_DIR}
                        echo 'Backup tarball extracted.'
                        EOF
                        """
                    }
                }
            }
        }
        stage('Stop PostgreSQL Service') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    sh """
                    ssh ubuntu@${EC2_HOST} <<'EOF'
                    set -e
                    echo 'Stopping PostgreSQL service...'
                    sudo systemctl stop postgresql
                    echo 'PostgreSQL service stopped.'
                    EOF
                    """
                }
            }
        }
        stage('Restore Data Directory') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    retry(2) {
                        sh """
                        ssh ubuntu@${EC2_HOST} <<'EOF'
                        set -e
                        echo 'Restoring data directory...'
                        sudo rm -rf ${DATA_DIR}/*
                        sudo cp -R ${RESTORE_DIR}/* ${DATA_DIR}
                        sudo chown -R postgres:postgres ${DATA_DIR}
                        echo 'Data directory replaced.'
                        EOF
                        """
                    }
                }
            }
        }
        stage('Start PostgreSQL Service') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    sh """
                    ssh ubuntu@${EC2_HOST} <<'EOF'
                    set -e
                    echo 'Starting PostgreSQL service...'
                    sudo systemctl start postgresql
                    echo 'PostgreSQL service started.'
                    EOF
                    """
                }
            }
        }
        stage('Verify Restoration') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    sh """
                    ssh ubuntu@${EC2_HOST} <<'EOF'
                    set -e
                    echo 'Verifying restoration...'
                    psql -U postgres -c 'SELECT 1;'
                    echo 'Database restoration verified.'
                    EOF
                    """
                }
            }
        }
        stage('Start DB-Sync Service') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    sh """
                    ssh ubuntu@${EC2_HOST} <<'EOF'
                    set -e
                    echo 'Restarting DB-Sync service...'
                    sudo systemctl restart cnode-dbsync.service
                    echo 'DB-Sync service has been restarted.'
                    EOF
                    """
                }
            }
        }
    }
    post {
        success {
            script {
                def message = [
                    channel: "${SLACK_CHANNEL}",
                    text: "Build Success: ${env.JOB_NAME} [${env.BUILD_NUMBER}] (<${env.BUILD_URL}|Open>)",
                    color: 'good',
                    icon_emoji: ':tada:',  // Emoji for success
                    username: 'Jenkins'  // Custom bot name
                ]
                // Send message to Slack
                httpRequest(
                    url: "${SLACK}",
                    httpMode: 'POST',
                    contentType: 'APPLICATION_JSON',
                    requestBody: groovy.json.JsonOutput.toJson(message)
                )
            }
        }
        failure {
            script {
                def message = [
                    channel: "${SLACK_CHANNEL}",
                    text: "Build Failed: ${env.JOB_NAME} [${env.BUILD_NUMBER}] (<${env.BUILD_URL}|Open>)",
                    color: 'danger',
                    icon_emoji: ':warning:',  // Emoji for failure
                    username: 'Jenkins'  // Custom bot name
                ]
                // Send message to Slack
                httpRequest(
                    url: "${SLACK}",
                    httpMode: 'POST',
                    contentType: 'APPLICATION_JSON',
                    requestBody: groovy.json.JsonOutput.toJson(message)
                )
            }
        }
        always {
            sshagent(credentials: ['SSH_KEY_CRED']) {
                sh """
                ssh ubuntu@${EC2_HOST} <<'EOF'
                set -e
                echo 'Cleaning up temporary files...'
                rm -rf ${RESTORE_DIR} /tmp/${BACKUP_FILE}.tar.gz
                echo 'Cleanup complete.'
                EOF
                """
            }
        }
    }
}
