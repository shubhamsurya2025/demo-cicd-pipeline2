pipeline {
    agent any
    parameters {
        choice(name: 'TARGET_ENV', choices: ['DEV', 'UAT'], description: 'Select deployment environment')
    }

    environment {
        DEV_HOST = '172.25.1.27'       // Replace with your actual DEV IP
        UAT_HOST = '172.25.1.28'       // Replace with your actual UAT IP
        REMOTE_DIR = '/opt/deployed-apps'
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
                    def host = (params.TARGET_ENV == 'DEV') ? env.DEV_HOST : env.UAT_HOST
                    def credId = (params.TARGET_ENV == 'DEV') ? 'ssh-server28-key' : 'ssh-server27-key'

                    echo "Deploying to ${params.TARGET_ENV} server at ${host}..."

                    def remote = [:]
                    remote.user = 'shubham'
                    remote.host = host
                    remote.allowAnyHosts = true

                    // SSH private key credential id configured in Jenkins credentials store
                    withCredentials([sshUserPrivateKey(credentialsId: credId, keyFileVariable: 'KEYFILE', usernameVariable: 'USER')]) {
                        remote.identity = KEYFILE

                        // Copy app.sh to remote server
                        sshPut remote: remote, from: 'app.sh', into: "${env.REMOTE_DIR}/app.sh"

                        // Run app.sh with sudo and capture output
                        def deployOut = sshCommand remote: remote, command: """
                            sudo chmod +x ${env.REMOTE_DIR}/app.sh
                            sudo ${env.REMOTE_DIR}/app.sh
                        """, returnStdout: true

                        echo "==== OUTPUT FROM ${params.TARGET_ENV} (${host}) ===="
                        echo deployOut.trim()

                        // Optional: Verify remote directory listing to confirm deployment
                        def lsOut = sshCommand remote: remote, command: "ls -l ${env.REMOTE_DIR}", returnStdout: true
                        echo "==== Directory listing on ${params.TARGET_ENV} (${host}) ===="
                        echo lsOut.trim()
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}
