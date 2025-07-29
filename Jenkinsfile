pipeline {
  agent any
  parameters {
    choice(name: 'DEPLOY_ENV', choices: ['DEV', 'UAT'], description: 'Where do you want to deploy?')
  }
  environment {
    JENKINS_HOME_DIR = "/var/lib/jenkins"
    UAT_DIR   = "/var/lib/jenkins/server-27-uat"
    DEV_DIR   = "/var/lib/jenkins/server-28-dev"
    WEB_ROOT  = "/var/lib/jenkins/userContent/demo-app"
    APP_URL   = "${JENKINS_URL}/userContent/demo-app/index.html"
  }
  stages {
    stage('System Diagnostics') {
      steps {
        sh '''
          echo "=========================================="
          echo "SYSTEM DIAGNOSTICS AND PERMISSIONS CHECK"
          echo "=========================================="
          echo "Current user: $(whoami)"
          echo "Current working directory: $(pwd)"
          echo "Current user ID: $(id)"
          echo "Jenkins Home Directory: ${JENKINS_HOME_DIR}"
          echo "Checking Jenkins home exists:"
          ls -ld "${JENKINS_HOME_DIR}" || echo "Jenkins home directory not found"
          echo "Checking Jenkins home permissions:"
          ls -la "${JENKINS_HOME_DIR}" | head -10
          echo "Checking if we can write to Jenkins home:"
          touch "${JENKINS_HOME_DIR}/test-write-permission" && echo "Write permission OK" && rm "${JENKINS_HOME_DIR}/test-write-permission" || echo "Write permission DENIED"
          echo "Available disk space:"
          df -h "${JENKINS_HOME_DIR}"
          echo "=========================================="
        '''
      }
    }
    stage('Prepare Folders - Verbose') {
      steps {
        sh '''
          echo "=========================================="
          echo "CREATING DIRECTORIES WITH DETAILED OUTPUT"
          echo "=========================================="
          set -x  # Enable command tracing
          
          echo "Step 1: Creating UAT directory with full path"
          mkdir -p -v "${UAT_DIR}/data" 2>&1 || { echo "FAILED to create UAT directory"; exit 1; }
          
          echo "Step 2: Creating DEV directory with full path"  
          mkdir -p -v "${DEV_DIR}/data" 2>&1 || { echo "FAILED to create DEV directory"; exit 1; }
          
          echo "Step 3: Creating WEB directory with full path"
          mkdir -p -v "${WEB_ROOT}" 2>&1 || { echo "FAILED to create WEB directory"; exit 1; }
          
          echo "Step 4: Setting permissions explicitly"
          chmod -v 755 "${UAT_DIR}" "${DEV_DIR}" "${WEB_ROOT}" 2>&1 || echo "Permission setting warning (non-critical)"
          
          echo "Step 5: Verifying directory creation"
          echo "UAT Directory:"
          ls -ld "${UAT_DIR}" && ls -la "${UAT_DIR}/data" || echo "UAT directory verification failed"
          echo "DEV Directory:"  
          ls -ld "${DEV_DIR}" && ls -la "${DEV_DIR}/data" || echo "DEV directory verification failed"
          echo "WEB Directory:"
          ls -ld "${WEB_ROOT}" || echo "WEB directory verification failed"
          
          echo "Step 6: Creating test files to verify write access"
          echo "test content" > "${UAT_DIR}/data/test.txt" && echo "UAT write test OK" || echo "UAT write test FAILED"
          echo "test content" > "${DEV_DIR}/data/test.txt" && echo "DEV write test OK" || echo "DEV write test FAILED"
          
          set +x  # Disable command tracing
          echo "=========================================="
        '''
      }
    }
    stage('Build Web App') {
      steps {
        script {
          def html = """
          <!DOCTYPE html><html><head>
            <meta charset='utf-8'><title>Demo – ${params.DEPLOY_ENV}</title>
          </head><body style='font-family:sans-serif;text-align:center;
                               padding-top:40px;background:#222;color:#fff'>
            <h1>CI/CD Demo ✔</h1>
            <h2>Environment: ${params.DEPLOY_ENV}</h2>
            <p>Build #${env.BUILD_NUMBER} – ${new Date()}</p>
            <p>Node: ${env.NODE_NAME}</p>
            <p>Workspace: ${env.WORKSPACE}</p>
          </body></html>
          """
          writeFile file: 'index.html', text: html
        }
        echo 'Web page created in workspace.'
      }
    }
    stage('Deploy App') {
      steps {
        script {
          def target = params.DEPLOY_ENV == 'DEV' ? env.DEV_DIR : env.UAT_DIR
          sh """
            echo "Deploying to: ${target}"
            echo "Copying index.html to target directory"
            cp -v index.html "${target}/" || { echo "Failed to copy to target"; exit 1; }
            echo "Copying index.html to web root"
            cp -v index.html "${WEB_ROOT}/" || { echo "Failed to copy to web root"; exit 1; }
            echo "Deployment completed successfully"
            echo "Target directory contents:"
            ls -la "${target}/" || echo "Target directory listing failed"
            echo "Web root contents:"
            ls -la "${WEB_ROOT}/" || echo "Web root listing failed"
          """
        }
      }
    }
    stage('Data Migration UAT → DEV') {
      when { expression { params.DEPLOY_ENV == 'DEV' } }
      steps {
        sh '''
          echo "=========================================="
          echo "DATA MIGRATION WITH DETAILED LOGGING"
          echo "=========================================="
          
          # Create sample data on UAT if missing
          if [ ! -f "${UAT_DIR}/data/customers.json" ]; then
            echo "Creating sample data on UAT server..."
            cat > "${UAT_DIR}/data/customers.json" <<EOF
{
  "customers": [
    {"id": 1, "name": "Alice", "tier": "gold"},
    {"id": 2, "name": "Bob", "tier": "silver"}
  ],
  "source": "UAT-Server-27",
  "created": "$(date -Iseconds)",
  "build": "${BUILD_NUMBER}",
  "workspace": "${WORKSPACE}",
  "node": "${NODE_NAME}"
}
EOF
            echo "Sample data created successfully"
          else
            echo "UAT data already exists"
          fi
          
          # Verify UAT data
          echo "UAT Source Data (${UAT_DIR}/data/customers.json):"
          cat "${UAT_DIR}/data/customers.json" || { echo "Failed to read UAT data"; exit 1; }
          
          # Migrate to DEV
          echo "Migrating data from UAT to DEV..."
          cp -v "${UAT_DIR}/data/customers.json" "${DEV_DIR}/data/customers.json" || { echo "Migration copy failed"; exit 1; }
          
          # Update source reference for DEV
          sed -i 's/UAT-Server-27/DEV-Server-28/' "${DEV_DIR}/data/customers.json"
          
          # Verify DEV data
          echo "DEV Destination Data (${DEV_DIR}/data/customers.json):"
          cat "${DEV_DIR}/data/customers.json" || { echo "Failed to read DEV data"; exit 1; }
          
          # Calculate and compare sizes
          UAT_SIZE=$(wc -c < "${UAT_DIR}/data/customers.json")
          DEV_SIZE=$(wc -c < "${DEV_DIR}/data/customers.json")
          
          echo "Migration Summary:"
          echo "  UAT file: ${UAT_DIR}/data/customers.json (${UAT_SIZE} bytes)"
          echo "  DEV file: ${DEV_DIR}/data/customers.json (${DEV_SIZE} bytes)"
          echo "  Status: SUCCESS"
          echo "=========================================="
        '''
      }
    }
  }
  post {
    always {
      script {
        echo """
        ==========================================
        🔍 TROUBLESHOOTING SUMMARY
        ==========================================
        Environment: ${params.DEPLOY_ENV}
        Build: #${env.BUILD_NUMBER}
        Node: ${env.NODE_NAME}
        Workspace: ${env.WORKSPACE}
        
        Expected Directory Locations:
        - UAT: /var/lib/jenkins/server-27-uat/data/
        - DEV: /var/lib/jenkins/server-28-dev/data/
        - Web: /var/lib/jenkins/userContent/demo-app/
        
        After this build, run these commands on Server 28:
        ssh shubham@172.25.1.28
        ls -la /var/lib/jenkins/server-27-uat/data/
        ls -la /var/lib/jenkins/server-28-dev/data/
        cat /var/lib/jenkins/server-27-uat/data/customers.json
        cat /var/lib/jenkins/server-28-dev/data/customers.json
        ==========================================
        """
      }
    }
    success {
      echo "✅ Pipeline completed successfully - directories should now exist!"
    }
    failure {
      echo "❌ Pipeline failed - check System Diagnostics stage for permission issues"
    }
  }
}
