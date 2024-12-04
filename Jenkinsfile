pipeline {
    agent any
    parameters {
        string(name: 'AWS_REGION', defaultValue: 'us-east-1', description: 'Region of the S3 bucket')
        string(name: 'EC2_REGION', defaultValue: 'us-east-1', description: 'Region of the EC2 instance')
        string(name: 'S3_BUCKET', defaultValue: 'statefile-remote-be', description: 'Name of the S3 bucket')
        string(name: 'EC2_HOST', defaultValue: 'ec2-98-80-51-171.compute-1.amazonaws.com', description: 'EC2 instance hostname or IP')
        string(name: 'NETWORK', defaultValue: 'Preview', description: 'Name of Instance we are restoring from')
        string(name: 'BACKUP_FILE', defaultValue: 'postgres_backup_2024', description: 'Name of the backup file to restore')
    }
    environment {
        RESTORE_DIR = "/tmp/postgres_restore"
        DATA_DIR = "/var/lib/postgresql/16/main/"
    }
    stages {
        stage('Check and Clean Directories') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    sh """
                    ssh ubuntu@${EC2_HOST} \
                    "if [ -d ${RESTORE_DIR} ] && [ \"$(ls -A ${RESTORE_DIR})\" ]; then rm -rf ${RESTORE_DIR}/*; echo 'Restore directory cleaned.'; fi"
                    ssh ubuntu@${EC2_HOST} \
                    "if [ -d ${DATA_DIR} ] && [ \"$(ls -A ${DATA_DIR})\" ]; then sudo rm -rf ${DATA_DIR}/*; echo 'Data directory cleaned.'; fi"
                    """
                }
            }
        }
        stage('Download Backup from S3') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    retry(3) {
                        sh """
                        ssh ubuntu@${EC2_HOST} \
                        "aws s3 cp s3://${S3_BUCKET}/${NETWORK}/${BACKUP_FILE}.tar.gz /tmp/ && echo 'Backup tarball downloaded.'"
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
                        ssh ubuntu@${EC2_HOST} \
                        "mkdir -p ${RESTORE_DIR} && tar -xzf /tmp/${BACKUP_FILE}.tar.gz -C ${RESTORE_DIR} && echo 'Backup tarball extracted.'"
                        """
                    }
                }
            }
        }
        stage('Stop PostgreSQL Service') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    sh """
                    ssh ubuntu@${EC2_HOST} \
                    "sudo systemctl stop postgresql && echo 'PostgreSQL service stopped.'"
                    """
                }
            }
        }
        stage('Restore Data Directory') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    retry(2) {
                        sh """
                        ssh ubuntu@${EC2_HOST} \
                        "sudo rm -rf ${DATA_DIR}/* && sudo cp -R ${RESTORE_DIR}/* ${DATA_DIR} && sudo chown -R postgres:postgres ${DATA_DIR} && echo 'Data directory replaced.'"
                        """
                    }
                }
            }
        }
        stage('Start PostgreSQL Service') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    sh """
                    ssh ubuntu@${EC2_HOST} \
                    "sudo systemctl start postgresql && echo 'PostgreSQL service started.'"
                    """
                }
            }
        }
        stage('Verify Restoration') {
            steps {
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    sh """
                    ssh ubuntu@${EC2_HOST} \
                    "psql -U postgres -c 'SELECT 1;' && echo 'Database restoration verified.'"
                    """
                }
            }
        }
        stage('Start DB-Sync Service') {
            steps{
                sshagent(credentials: ['SSH_KEY_CRED']) {
                    sh """
                    ssh ubuntu@${EC2_HOST} \
                    "sudo systemctl restart cnode-dbsync.service && echo 'DB-Sync service has been restarted'"
                    """
                }
            }
        }
    }
    post {
        always {
            sshagent(credentials: ['SSH_KEY_CRED']) {
                sh "ssh ubuntu@${EC2_HOST} 'rm -rf ${RESTORE_DIR} /tmp/${BACKUP_FILE}.tar.gz'"
            }
        }
    }
}
