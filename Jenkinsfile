pipeline {
    agent any
    parameters {
        string(name: 'BACKUP_FILE', defaultValue: 'postgres_backup_2024', description: 'Name of the backup file to restore')
    }
    environment {
        RESTORE_DIR = "/tmp/postgres_restore"
        DATA_DIR = "/opt/cardano/cnode/guild-db/pgdb/"
        SLACK_CHANNEL = '#jenkins-notifications'
        LEDGER_DIR = "/opt/cardano/cnode/db"
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
                    string(credentialsId: 'IAM_ROLE', variable: 'IAM_ROLE'),
                    string(credentialsId: 'PG_PASS', variable: 'PG_PASS')
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
                        env.PG_PASS = "${PG_PASS}"
                    }
                }
            }
        }
        stage('Check and Clean Directories') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    sh """
                    ssh ubuntu@${EC2_HOST} 'mkdir -p \\${RESTORE_DIR} && \
                    if [ -d \\${RESTORE_DIR} ] && [ "\\\$(ls -A \\${RESTORE_DIR})" ]; then \
                        rm -rf \\${RESTORE_DIR}/* && echo "Restore directory cleaned.";\
                    fi && \
                    if [ -d \\${DATA_DIR} ] && [ "\\\$(ls -A \\${DATA_DIR})" ]; then \
                        sudo rm -rf \\${DATA_DIR}/* && echo "Data directory cleaned.";\
                    fi'
                    """
                }
            }
        }
        stage('Download Backup from S3') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    retry(3) {
                        sh """
                        ssh ubuntu@${EC2_HOST} 'aws s3 cp s3://\\${S3_BUCKET}/\\${NETWORK}/\\${BACKUP_FILE}.tar.gz /tmp/ && echo "Backup tarball downloaded."'
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
                        ssh ubuntu@${EC2_HOST} 'sudo mkdir -p \\${RESTORE_DIR} && \
                        sudo tar -xzf /tmp/\\${BACKUP_FILE}.tar.gz -C \\${RESTORE_DIR} && echo "Backup tarball extracted."'
                        """
                    }
                }
            }
        }
        stage('Stop DB-Sync Service') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    sh """
                    ssh ubuntu@${EC2_HOST} 'sudo systemctl stop cnode-dbsync.service && echo "DB-Sync service stopped."'
                    """
                }
            }
        }
        stage('Create DB') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    sh """
                    ssh ubuntu@${EC2_HOST} 'export PGPASSFILE=${PG_PASS} && ./git/cardano-db-sync/scripts/postgresql-setup.sh --createdb'
                    """
                }
            }
        }
        stage('Stop PostgreSQL Service') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    sh """
                    ssh ubuntu@${EC2_HOST} 'sudo systemctl stop postgresql && echo "PostgreSQL service stopped."'
                    """
                }
            }
        }
        stage('Restore Data Directory') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    retry(2) {
                        sh """
                        ssh ubuntu@${EC2_HOST} 'sudo rm -rf \\${DATA_DIR}/* && \
                        sudo cp \\${RESTORE_DIR}/backup_manifest \\${DATA_DIR} && \
                        sudo tar -xzf \\${RESTORE_DIR}/base.tar.gz -C \\${DATA_DIR} && \
                        sudo tar -xzf \\${RESTORE_DIR}/pg_wal.tar.gz -C \\${DATA_DIR}/pg_wal && \
                        sudo chown -R postgres:postgres \\${DATA_DIR} && echo "Data directory replaced." && \
                        sudo rm -rf \\${LEDGER_DIR}/* && \
                        sudo cp -r \\${RESTORE_DIR}/gsm \\${LEDGER_DIR}/ && \
                        sudo cp -r \\${RESTORE_DIR}/immutable \\${LEDGER_DIR}/ && \
                        sudo cp -r \\${RESTORE_DIR}/ledger \\${LEDGER_DIR}/ && \
                        sudo cp \\${RESTORE_DIR}/lock \\${LEDGER_DIR}/ && \
                        sudo cp \\${RESTORE_DIR}/protocolMagicId \\${LEDGER_DIR}/ && \
                        sudo cp -r \\${RESTORE_DIR}/volatile \\${LEDGER_DIR}/ && \
                        chown -R ubuntu:ubuntu \\${LEDGER_DIR} '
                        """
                    }
                }
            }
        }
        stage('Start Node') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    sh """
                    ssh ubuntu@${EC2_HOST} 'sudo systemctl restart cnode.service && \
                    sleep 60'
                    """
                }
            }
        }
        stage('Start PostgreSQL Service') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    sh """
                    ssh ubuntu@${EC2_HOST} 'sudo systemctl restart postgresql && echo "PostgreSQL service started."'
                    """
                }
            }
        }
        stage('Verify Restoration') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    sh """
                    ssh ubuntu@${EC2_HOST} 'psql postgres -c "SELECT 1;" && echo "Database restoration verified."'
                    """
                }
            }
        }
        stage('Start DB-Sync Service') {
            steps{
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    sh """
                    ssh ubuntu@${EC2_HOST} 'sudo systemctl restart cnode-dbsync.service && echo "DB-Sync service has been restarted."'
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
                sh "ssh ubuntu@${EC2_HOST} 'rm -rf \\${RESTORE_DIR} /tmp/\\${BACKUP_FILE}.tar.gz'"
            }
        }
    }
}
