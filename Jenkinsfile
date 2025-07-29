pipeline {
  agent any

  parameters {
    choice(name: 'DEPLOY_ENV',
           choices: ['DEV', 'UAT'],
           description: 'Where to deploy: DEV → 172.25.1.28, UAT → 172.25.1.27')
  }

  environment {
    JENKINS_HOME_DIR = '/var/lib/jenkins'
    UAT_IP           = '172.25.1.27'
    DEV_IP           = '172.25.1.28'
    UAT_DIR          = "${JENKINS_HOME_DIR}/${UAT_IP}/data"
    DEV_DIR          = "${JENKINS_HOME_DIR}/${DEV_IP}/data"
    WEB_ROOT         = "${JENKINS_HOME_DIR}/userContent/demo-app"
    APP_URL          = "http://${UAT_IP}:8181/userContent/demo-app/index.html"
  }

  stages {
    stage('Debug Environment') {
      steps {
        sh '''
          echo "=========================================="
          echo "JENKINS ENVIRONMENT DEBUG"
          echo "=========================================="
          echo "Workspace: ${WORKSPACE}"
          echo "JENKINS_HOME: ${JENKINS_HOME_DIR}"
          echo "UAT_DIR: ${UAT_DIR}"
          echo "DEV_DIR: ${DEV_DIR}"
          echo "WEB_ROOT: ${WEB_ROOT}"
          echo "Current user: $(whoami)"
          echo "=========================================="
        '''
      }
    }

    stage('Prepare Folders') {
      steps {
        sh '''
          echo "Creating directories with real IP names..."
          mkdir -p "${UAT_DIR}" "${DEV_DIR}" "${WEB_ROOT}"
          chmod 755 "${JENKINS_HOME_DIR}/${UAT_IP}" "${JENKINS_HOME_DIR}/${DEV_IP}" "${WEB_ROOT}" 2>/dev/null || echo "Permission setting skipped"
          
          echo "Verification - Directory listing:"
          ls -ld "${JENKINS_HOME_DIR}/${UAT_IP}" "${JENKINS_HOME_DIR}/${DEV_IP}" "${WEB_ROOT}" || echo "Some directories missing"
          
          echo "UAT directory contents:"
          ls -la "${UAT_DIR}" || echo "UAT directory empty"
          echo "DEV directory contents:"
          ls -la "${DEV_DIR}" || echo "DEV directory empty"
        '''
      }
    }

    stage('Build Web App') {
      steps {
        script {
          def html = """
          <!DOCTYPE html>
          <html><head>
            <meta charset='utf-8'>
            <title>CI/CD Demo – ${params.DEPLOY_ENV}</title>
          </head>
          <body style='font-family:sans-serif;text-align:center;padding-top:40px;background:#f0f0f0;'>
            <h1>🚀 CI/CD Demo Application</h1>
            <h2>Environment: ${params.DEPLOY_ENV}</h2>
            <p><strong>Build #${env.BUILD_NUMBER}</strong></p>
            <p>Timestamp: ${new Date()}</p>
            <p>Target Server: ${params.DEPLOY_ENV == 'DEV' ? env.DEV_IP : env.UAT_IP}</p>
            <p style='color:green;'>✅ Pipeline Deployment Successful!</p>
          </body></html>
          """
          writeFile file: 'index.html', text: html
        }
        echo 'Web application built successfully in workspace.'
      }
    }

    stage('Deploy App') {
      steps {
        script {
          def targetDir = (params.DEPLOY_ENV == 'DEV') ? env.DEV_DIR : env.UAT_DIR
          def targetServer = (params.DEPLOY_ENV == 'DEV') ? env.DEV_IP : env.UAT_IP
          
          sh """
            echo "Deploying to ${params.DEPLOY_ENV} environment (${targetServer})"
            cp -v index.html "${targetDir}/"
            cp -v index.html "${WEB_ROOT}/"
            echo "Deployment completed successfully to ${targetDir}"
            
            echo "Target directory contents:"
            ls -la "${targetDir}/" || echo "Target directory listing failed"
            echo "Web root contents:"
            ls -la "${WEB_ROOT}/" || echo "Web root listing failed"
          """
          
          echo "🌍 Application URL: ${env.APP_URL}"
        }
      }
    }

    stage('Data Migration UAT → DEV') {
      when { expression { params.DEPLOY_ENV == 'DEV' } }
      steps {
        sh '''
          echo "=========================================="
          echo "STARTING DATA MIGRATION: UAT → DEV"
          echo "Source: ${UAT_IP} (${UAT_DIR})"
          echo "Destination: ${DEV_IP} (${DEV_DIR})"
          echo "=========================================="
          
          # Create sample data on UAT if missing
          if [ ! -f "${UAT_DIR}/data.json" ]; then
            echo "Creating sample data on UAT server (${UAT_IP})..."
            cat > "${UAT_DIR}/data.json" <<EOF
{
  "customers": [
    {"id": 1, "name": "Alice", "tier": "gold"},
    {"id": 2, "name": "Bob", "tier": "silver"}
  ],
  "source": "UAT-Server-${UAT_IP}",
  "created": "$(date -Iseconds)",
  "build": "${BUILD_NUMBER}"
}
EOF
            echo "Sample data created on UAT"
          else
            echo "UAT data already exists"
          fi

          # Show source data
          echo "UAT Source Data (${UAT_IP}):"
          cat "${UAT_DIR}/data.json" || { echo "Failed to read UAT data"; exit 1; }

          # Migrate to DEV
          echo "Migrating data from UAT (${UAT_IP}) to DEV (${DEV_IP})..."
          cp -v "${UAT_DIR}/data.json" "${DEV_DIR}/data.json" || { echo "Migration failed"; exit 1; }
          
          # Update source reference for DEV
          sed -i "s/UAT-Server-${UAT_IP}/DEV-Server-${DEV_IP}/" "${DEV_DIR}/data.json"
          
          # Verify migration
          echo "DEV Destination Data (${DEV_IP}):"
          cat "${DEV_DIR}/data.json" || { echo "Failed to read DEV data"; exit 1; }
          
          # Calculate sizes
          UAT_SIZE=$(wc -c < "${UAT_DIR}/data.json")
          DEV_SIZE=$(wc -c < "${DEV_DIR}/data.json")
          
          echo "Migration Summary:"
          echo "  UAT file (${UAT_IP}): ${UAT_SIZE} bytes"
          echo "  DEV file (${DEV_IP}): ${DEV_SIZE} bytes"
          echo "  Status: ✅ SUCCESS"
          echo "=========================================="
        '''
      }
    }

    stage('Verify Migration') {
      steps {
        script {
          // Use shell commands instead of fileExists to avoid plugin dependency
          def uatExists = sh(script: "test -f '${UAT_DIR}/data.json'", returnStatus: true) == 0
          def devExists = sh(script: "test -f '${DEV_DIR}/data.json'", returnStatus: true) == 0
          
          if (uatExists) {
            echo "✅ UAT data.json exists at ${env.UAT_DIR}/data.json"
          } else {
            echo "⚠️ UAT data.json missing (may not have been created yet)"
          }
          
          if (params.DEPLOY_ENV == 'DEV') {
            if (devExists) {
              echo "✅ DEV data.json exists at ${env.DEV_DIR}/data.json"
              echo "✅ Data migration UAT → DEV completed successfully"
            } else {
              error "❌ DEV data.json missing - migration failed"
            }
          }
        }
        
        sh '''
          echo "=========================================="
          echo "FINAL VERIFICATION SUMMARY"
          echo "=========================================="
          echo "Directory structure:"
          find ${JENKINS_HOME_DIR}/${UAT_IP} ${JENKINS_HOME_DIR}/${DEV_IP} -type f 2>/dev/null || echo "Some directories not found"
          echo "Web application:"
          ls -la "${WEB_ROOT}/" || echo "Web directory not accessible"
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
        ✅ PIPELINE COMPLETED SUCCESSFULLY!
        ==========================================
        Environment: ${params.DEPLOY_ENV}
        Target Server: ${params.DEPLOY_ENV == 'DEV' ? env.DEV_IP : env.UAT_IP}
        
        🌍 Application URL: ${env.APP_URL}
        
        Data Migration: ${params.DEPLOY_ENV == 'DEV' ? '✅ Completed (UAT → DEV)' : '⏸️ Not applicable for UAT deployment'}
        
        Verification Commands:
        ssh shubham@${env.DEV_IP}
        ls -la /var/lib/jenkins/${env.UAT_IP}/data/
        ls -la /var/lib/jenkins/${env.DEV_IP}/data/
        cat /var/lib/jenkins/${env.UAT_IP}/data/data.json
        cat /var/lib/jenkins/${env.DEV_IP}/data/data.json
        ==========================================
        """
      }
    }
    failure {
      echo """
      ==========================================
      ❌ PIPELINE FAILED
      ==========================================
      Check the console output above for details.
      Common issues:
      1. Directory creation permissions
      2. File system access problems
      3. Jenkins workspace issues
      ==========================================
      """
    }
  }
}
