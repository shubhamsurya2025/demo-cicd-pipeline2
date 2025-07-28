pipeline {
    agent any
    parameters {
        choice(name: 'TARGET_ENV', choices: ['DEV', 'UAT'], description: 'Select deployment environment')
    }
    environment {
        DEV_HOST    = '172.25.1.28'       // DEV = Server 28
        UAT_HOST    = '172.25.1.27'       // UAT = Server 27
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
                echo "Building application..."
                sh 'chmod +x app.sh'
                sh './app.sh'
            }
        }
        stage('Deploy & Capture Output') {
            steps {
                script {
                    // Select host & credential based on TARGET_ENV
                    def host   = (params.TARGET_ENV == 'DEV') ? env.DEV_HOST : env.UAT_HOST
                    def credId = (params.TARGET_ENV == 'DEV') ? 'ssh-server28-key' : 'ssh-server27-key'
                    echo "Deploying to ${params.TARGET_ENV} at ${host}…"

                    def remote = [
                        user:          'shubham',
                        host:          host,
                        allowAnyHosts: true
                    ]
                    withCredentials([sshUserPrivateKey(credentialsId: credId,
                                                       keyFileVariable: 'KEYFILE',
                                                       usernameVariable: 'USER')]) {
                        remote.identity = KEYFILE
                        // Copy and execute
                        sshPut    remote: remote, from: 'app.sh', into: "${env.REMOTE_DIR}/app.sh"
                        def out = sshCommand remote: remote, command: """
                            sudo chmod +x ${env.REMOTE_DIR}/app.sh
                            sudo ${env.REMOTE_DIR}/app.sh
                        """, returnStdout: true
                        echo "=== OUTPUT FROM ${params.TARGET_ENV} (${host}) ==="
                        echo out.trim()

                        // Verify directory listing
                        def ls = sshCommand remote: remote, command: "ls -l ${env.REMOTE_DIR}", returnStdout: true
                        echo "=== Directory listing on ${params.TARGET_ENV} (${host}) ==="
                        echo ls.trim()
                    }
                }
            }
        }
        stage('Data Migration UAT→DEV') {
            when { expression { params.TARGET_ENV == 'DEV' } }
            steps {
                script {
                    echo "Migrating data.txt from UAT → DEV…"

                    // Define remotes
                    def uatRemote = [ user: 'shubham', host: env.UAT_HOST, allowAnyHosts: true ]
                    def devRemote = [ user: 'shubham', host: env.DEV_HOST, allowAnyHosts: true ]

                    withCredentials([
                        sshUserPrivateKey(credentialsId: 'ssh-server27-key', keyFileVariable: 'UAT_KEY', usernameVariable: 'USR1'),
                        sshUserPrivateKey(credentialsId: 'ssh-server28-key', keyFileVariable: 'DEV_KEY', usernameVariable: 'USR2')
                    ]) {
                        uatRemote.identity = UAT_KEY
                        devRemote.identity = DEV_KEY

                        // Pull from UAT into workspace
                        sshGet  remote: uatRemote, from: "${env.REMOTE_DIR}/data.txt", into: 'data.txt'
                        // Copy into DEV home or /tmp to avoid sudo
                        sshPut  remote: devRemote, from: 'data.txt', into: '/tmp/data.txt'
                        // Move into final directory with sudo
                        sshCommand remote: devRemote, command: 'sudo mv /tmp/data.txt ' + env.REMOTE_DIR
                        // Verify content
                        def dataOut = sshCommand remote: devRemote, command: "cat ${env.REMOTE_DIR}/data.txt", returnStdout: true
                        echo "=== MIGRATED data.txt CONTENT ON DEV ==="
                        echo dataOut.trim()
                    }
                }
            }
        }
    }
    post {
        success { echo "Pipeline completed successfully!" }
        failure { echo "Pipeline failed - check logs for details." }
    }
}
