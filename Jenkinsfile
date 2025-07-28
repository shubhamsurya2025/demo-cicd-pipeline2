pipeline {
    agent any
    parameters {
        choice(name: 'TARGET_ENV', choices: ['DEV', 'UAT'], description: 'Select deployment environment')
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
        stage('Deploy') {
            steps {
                script {
                    if (params.TARGET_ENV == 'DEV') {
                        echo "Deploying to Server 28 (DEV)..."
                        sh '''
                            echo "Deployment to DEV server completed"
                            echo "Environment: DEV"
                            echo "Target: Server 28"
                        '''
                    } else {
                        echo "Deploying to Server 27 (UAT)..."
                        sh '''
                            echo "Deployment to UAT server completed"
                            echo "Environment: UAT"
                            echo "Target: Server 27"
                        '''
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
