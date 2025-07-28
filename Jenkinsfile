pipeline {
  agent any

  parameters {
    choice(name: 'DEPLOY_ENV',
           choices: ['DEV', 'UAT'],
           description: 'Where do you want to deploy?')
  }

  environment {
    JENKINS_HOME_DIR = "/var/lib/jenkins"
    UAT_DIR   = "/var/lib/jenkins/server-27-uat"
    DEV_DIR   = "/var/lib/jenkins/server-28-dev"
    WEB_ROOT  = "/var/lib/jenkins/userContent/demo-app"
    APP_URL   = "${JENKINS_URL}/userContent/demo-app/index.html"
  }

  stages {

    stage('Debug Environment') {
      steps {
        sh '''
          echo "=========================================="
          echo "DEBUGGING JENKINS ENVIRONMENT"
          echo "=========================================="
          echo "Current working directory: $(pwd)"
          echo "JENKINS_HOME: ${JENKINS_HOME_DIR}"
          echo "UAT_DIR: ${UAT_DIR}"
          echo "DEV_DIR: ${DEV_DIR}"
          echo "WEB_ROOT: ${WEB_ROOT}"
          echo "Workspace: ${WORKSPACE}"
          echo "Current user: $(whoami)"
          echo "Jenkins user check: $(id jenkins 2>/dev/null || echo 'Jenkins user not found')"
          echo "=========================================="
        '''
      }
    }

    stage('Prepare Folders') {
      steps {
        sh '''
          echo "=========================================="
          echo "CREATING SIMULATION DIRECTORIES"
          echo "=========================================="
          
          # Create directories with explicit paths and verbose output
          echo "Creating UAT directory: ${UAT_DIR}/data"
          mkdir -p "${UAT_DIR}/data" || { echo "Failed to create UAT dir"; exit 1; }
          
          echo "Creating DEV directory: ${DEV_DIR}/data"
          mkdir -p "${DEV_DIR}/data" || { echo "Failed to create DEV dir"; exit 1; }
          
          echo "Creating WEB directory: ${WEB_ROOT}"
          mkdir -p "${WEB_ROOT}" || { echo "Failed to create WEB dir"; exit 1; }
          
          echo "Setting permissions..."
          chmod 755 "${UAT_DIR}" "${DEV_DIR}" "${WEB_ROOT}" 2>/dev/null || echo "Permission setting skipped"
          
          echo "Verification - Directory listing:"
          ls -ld "${UAT_DIR}" "${DEV_DIR}" "${WEB_ROOT}" || echo "Some directories missing"
          
          echo "Content check:"
          ls -la "${UAT_DIR}/data" || echo "UAT data dir empty/missing"
          ls -la "${DEV_DIR}/data" || echo "DEV data dir empty/missing"
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
            <p>Data Migration: ${params.DEPLOY_ENV == 'DEV' ? 'ENABLED' : 'DISABLED'}</p>
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
            cp index.html "${target}/" || { echo "Failed to copy to target"; exit 1; }
            cp index.html "${WEB_ROOT}/" || { echo "Failed to copy to web root"; exit 1; }
            echo "Deployment completed successfully"
            ls -la "${target}/" || echo "Target directory listing failed"
          """
        }
      }
    }

    stage('Data Migration UAT → DEV') {
      when { expression { params.DEPLOY_ENV == 'DEV' } }
      steps {
        sh '''
          echo "=========================================="
          echo "STARTING DATA MIGRATION"
          echo "=========================================="
          
          # Create sample data on UAT if missing
          if [ ! -f "${UAT_DIR}/data/customers.json" ]; then
            echo "Creating sample data on UAT..."
            cat > "${UAT_DIR}/data/customers.json" <<EOF
{
  "customers": [
    {"id": 1, "name": "Alice", "tier": "gold"},
    {"id": 2, "name": "Bob", "tier": "silver"}
  ],
  "source": "UAT-Server-27",
  "created": "$(date -Iseconds)",
  "build": "${BUILD_NUMBER}"
}
EOF
            echo "Sample data created on UAT"
          else
            echo "UAT data already exists"
          fi

          # Show source data
          echo "UAT Source Data:"
          cat "${UAT_DIR}/data/customers.json" || { echo "Failed to read UAT data"; exit 1; }

          # Migrate to DEV
          echo "Migrating data from UAT to DEV..."
          cp "${UAT_DIR}/data/customers.json" "${DEV_DIR}/data/customers.json" || { echo "Migration failed"; exit 1; }
          
          # Update source reference for DEV
          sed -i 's/UAT-Server-27/DEV-Server-28/' "${DEV_DIR}/data/customers.json"
          
          # Verify migration
          echo "DEV Destination Data:"
          cat "${DEV_DIR}/data/customers.json" || { echo "Failed to read DEV data"; exit 1; }
          
          # Calculate size
          UAT_SIZE=$(wc -c < "${UAT_DIR}/data/customers.json")
          DEV_SIZE=$(wc -c < "${DEV_DIR}/data/customers.json")
          
          echo "Migration Summary:"
          echo "  UAT file size: ${UAT_SIZE} bytes"
          echo "  DEV file size: ${DEV_SIZE} bytes"
          echo "  Status: SUCCESS"
          echo "=========================================="
        '''
      }
    }

    stage('Verification') {
      steps {
        sh '''
          echo "=========================================="
          echo "FINAL VERIFICATION"
          echo "=========================================="
          
          echo "Directory Structure:"
          find /var/lib/jenkins/server-* -type f 2>/dev/null || echo "No server directories found"
          
          echo "Web Application:"
          ls -la "${WEB_ROOT}/" || echo "Web directory not found"
          
          echo "Application URL: ${APP_URL}"
          echo "=========================================="
        '''
      }
    }
  }

  post {
    success {
      script {
        echo """
        ==========================================
        ✅ PIPELINE COMPLETED SUCCESSFULLY
        ==========================================
        Environment: ${params.DEPLOY_ENV}
        Web URL: ${env.APP_URL}
        Build: #${env.BUILD_NUMBER}
        ==========================================
        """
      }
    }
    failure {
      echo """
      ==========================================
      ❌ PIPELINE FAILED
      ==========================================
      Check the console output above for errors.
      Most likely issues:
      1. Permission problems creating directories
      2. Jenkins workspace issues
      3. Missing Jenkins home directory access
      ==========================================
      """
    }
  }
}
