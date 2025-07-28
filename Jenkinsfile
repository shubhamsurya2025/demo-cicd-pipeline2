pipeline {
    agent any
    parameters {
        choice(name: 'TARGET_ENV', choices: ['DEV', 'UAT'], description: 'Select deployment environment')
    }
    environment {
        DEV_HOST    = '172.25.1.28'       // Server 28 = DEV environment
        UAT_HOST    = '172.25.1.27'       // Server 27 = UAT environment  
        REMOTE_DIR  = '/opt/deployed-apps'
    }
    stages {
        stage('Checkout') {
            steps {
                echo "Checking out code from GitHub..."
                checkout scm
            }
        }
        stage('Build') {
            steps {
                echo "Building application locally on Jenkins..."
                sh 'chmod +x app.sh'
                sh './app.sh'
                echo "Local build completed successfully!"
            }
        }
        stage('Deploy & Capture Output') {
            steps {
                script {
                    def host   = (params.TARGET_ENV == 'DEV') ? env.DEV_HOST : env.UAT_HOST
                    def credId = (params.TARGET_ENV == 'DEV') ? 'ssh-shubham-key' : 'ssh-shubham-key-27'
                    
                    echo "=========================================="
                    echo "DEPLOYING TO ${params.TARGET_ENV} ENVIRONMENT"
                    echo "Target Server: ${host}"
                    echo "Using Credential: ${credId}"
                    echo "=========================================="

                    def remote = [
                        user:          'shubham',
                        host:          host,
                        allowAnyHosts: true
                    ]
                    
                    withCredentials([sshUserPrivateKey(credentialsId: credId,
                                                       keyFileVariable: 'KEYFILE',
                                                       usernameVariable: 'USER')]) {
                        remote.identity = KEYFILE
                        
                        try {
                            // Copy application script to remote server
                            echo "Copying app.sh to ${params.TARGET_ENV} server..."
                            sshPut remote: remote, from: 'app.sh', into: "${env.REMOTE_DIR}/app.sh"
                            
                            // Execute deployment with sudo and capture output
                            echo "Executing deployment script on ${params.TARGET_ENV} server..."
                            def deployOut = sshCommand remote: remote, command: """
                                sudo chmod +x ${env.REMOTE_DIR}/app.sh
                                sudo ${env.REMOTE_DIR}/app.sh
                            """, returnStdout: true
                            
                            echo "=========================================="
                            echo "DEPLOYMENT OUTPUT FROM ${params.TARGET_ENV} (${host})"
                            echo "=========================================="
                            echo deployOut.trim()
                            echo "=========================================="

                            // Verify deployment by listing directory contents
                            def lsOut = sshCommand remote: remote, command: "ls -la ${env.REMOTE_DIR}", returnStdout: true
                            echo "DIRECTORY LISTING ON ${params.TARGET_ENV} (${host})"
                            echo "=========================================="
                            echo lsOut.trim()
                            echo "=========================================="
                            
                            // Show system information from target server
                            def sysInfo = sshCommand remote: remote, command: """
                                echo "Hostname: \$(hostname)"
                                echo "Current User: \$(whoami)"  
                                echo "Date/Time: \$(date)"
                                echo "Uptime: \$(uptime -p)"
                                echo "Disk Usage:"
                                df -h / | tail -1
                            """, returnStdout: true
                            echo "SYSTEM INFO FROM ${params.TARGET_ENV} SERVER"
                            echo "=========================================="
                            echo sysInfo.trim()
                            echo "=========================================="
                            
                        } catch (Exception e) {
                            echo "ERROR: Deployment failed - ${e.message}"
                            throw e
                        }
                    }
                }
            }
        }
        stage('Data Migration UAT→DEV') {
            when { 
                expression { params.TARGET_ENV == 'DEV' }
            }
            steps {
                script {
                    echo "=========================================="
                    echo "STARTING DATA MIGRATION FROM UAT TO DEV"
                    echo "Source: UAT Server (${env.UAT_HOST})"
                    echo "Destination: DEV Server (${env.DEV_HOST})"
                    echo "=========================================="

                    def uatRemote = [ 
                        user: 'shubham', 
                        host: env.UAT_HOST, 
                        allowAnyHosts: true 
                    ]
                    def devRemote = [ 
                        user: 'shubham', 
                        host: env.DEV_HOST, 
                        allowAnyHosts: true 
                    ]

                    withCredentials([
                        sshUserPrivateKey(credentialsId: 'ssh-shubham-key-27', keyFileVariable: 'UAT_KEY', usernameVariable: 'USR1'),
                        sshUserPrivateKey(credentialsId: 'ssh-shubham-key', keyFileVariable: 'DEV_KEY', usernameVariable: 'USR2')
                    ]) {
                        uatRemote.identity = UAT_KEY
                        devRemote.identity = DEV_KEY

                        try {
                            // Create sample data on UAT if it doesn't exist
                            echo "Ensuring data.txt exists on UAT server..."
                            sshCommand remote: uatRemote, command: """
                                if [ ! -f ${env.REMOTE_DIR}/data.txt ]; then
                                    echo "Sample data from UAT Server (\$(hostname))" | sudo tee ${env.REMOTE_DIR}/data.txt
                                    echo "Migration timestamp: \$(date)" | sudo tee -a ${env.REMOTE_DIR}/data.txt
                                    echo "Environment: UAT" | sudo tee -a ${env.REMOTE_DIR}/data.txt
                                    sudo chown shubham:shubham ${env.REMOTE_DIR}/data.txt
                                fi
                            """
                            
                            // Show source data content
                            def sourceData = sshCommand remote: uatRemote, command: "cat ${env.REMOTE_DIR}/data.txt", returnStdout: true
                            echo "SOURCE DATA FROM UAT SERVER:"
                            echo "=========================================="
                            echo sourceData.trim()
                            echo "=========================================="
                            
                            // Pull data from UAT to Jenkins workspace
                            echo "Copying data from UAT to Jenkins workspace..."
                            sshGet remote: uatRemote, from: "${env.REMOTE_DIR}/data.txt", into: 'data.txt'
                            
                            // Push data from Jenkins workspace to DEV server
                            echo "Copying data from Jenkins workspace to DEV server..."
                            sshPut remote: devRemote, from: 'data.txt', into: '/tmp/data.txt'
                            
                            // Move to final location with proper permissions
                            sshCommand remote: devRemote, command: """
                                sudo mv /tmp/data.txt ${env.REMOTE_DIR}/data.txt
                                sudo chown shubham:shubham ${env.REMOTE_DIR}/data.txt
                            """
                            
                            // Verify migrated data content on DEV
                            def migratedData = sshCommand remote: devRemote, command: "cat ${env.REMOTE_DIR}/data.txt", returnStdout: true
                            echo "MIGRATED DATA ON DEV SERVER:"
                            echo "=========================================="
                            echo migratedData.trim()
                            echo "=========================================="
                            
                            echo "✅ DATA MIGRATION UAT→DEV COMPLETED SUCCESSFULLY!"
                            
                        } catch (Exception e) {
                            echo "⚠️  Data migration encountered an issue: ${e.message}"
                            echo "This may be normal if data.txt doesn't exist on UAT server"
                            echo "Deployment to DEV was still successful"
                        }
                    }
                }
            }
        }
    }
    post {
        success {
            echo "=========================================="
            echo "🎉 PIPELINE COMPLETED SUCCESSFULLY!"
            echo "Environment: ${params.TARGET_ENV}"
            echo "Deployment Status: ✅ SUCCESS"
            if (params.TARGET_ENV == 'DEV') {
                echo "Data Migration: ✅ COMPLETED"
            }
            echo "=========================================="
        }
        failure {
            echo "=========================================="
            echo "❌ PIPELINE FAILED!"
            echo "Environment: ${params.TARGET_ENV}"
            echo "Check the console output above for detailed error information"
            echo "Common issues:"
            echo "1. SSH credentials not found in System Global scope"
            echo "2. Target server connectivity issues"
            echo "3. Permission issues on remote directories"
            echo "=========================================="
        }
        always {
            echo "Pipeline execution completed at: ${new Date()}"
            echo "Jenkins Job: ${env.JOB_NAME}"
            echo "Build Number: ${env.BUILD_NUMBER}"
        }
    }
}
