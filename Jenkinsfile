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
    APP_URL_UAT      = "http://${UAT_IP}:8181/userContent/demo-app/index.html"
    APP_URL_DEV      = "http://${UAT_IP}:8181/userContent/demo-app/index.html"
  }

  tools {
    // Ensure Pipeline Utility Steps plugin is available
    // to use fileExists and findFiles
  }

  stages {
    stage('Debug Environment') {
      steps {
        sh '''
          echo "Workspace: ${WORKSPACE}"
          echo "JENKINS_HOME: ${JENKINS_HOME_DIR}"
          echo "UAT_DIR: ${UAT_DIR}"
          echo "DEV_DIR: ${DEV_DIR}"
          echo "WEB_ROOT: ${WEB_ROOT}"
        '''
      }
    }

    stage('Prepare Folders') {
      steps {
        sh '''
          mkdir -p "${UAT_DIR}" "${DEV_DIR}" "${WEB_ROOT}"
          chmod 755 "${JENKINS_HOME_DIR}" "${UAT_DIR}" "${DEV_DIR}" "${WEB_ROOT}"
          echo "Directories ready:"
          ls -ld "${JENKINS_HOME_DIR}/${UAT_IP}" "${JENKINS_HOME_DIR}/${DEV_IP}" "${WEB_ROOT}"
        '''
      }
    }

    stage('Build Web App') {
      steps {
        script {
          def html = """
          <!DOCTYPE html>
          <html><head><meta charset='utf-8'><title>Demo – ${params.DEPLOY_ENV}</title></head>
          <body style='font-family:sans-serif;text-align:center;padding-top:40px;'>
            <h1>CI/CD Demo</h1>
            <p>Environment: ${params.DEPLOY_ENV}</p>
            <p>Build #${env.BUILD_NUMBER}</p>
          </body></html>
          """
          writeFile file: 'index.html', text: html
        }
      }
    }

    stage('Deploy App') {
      steps {
        script {
          def target = (params.DEPLOY_ENV == 'DEV') ? DEV_DIR : UAT_DIR
          sh """
            cp -v index.html "${target}/"
            cp -v index.html "${WEB_ROOT}/"
            echo "App deployed to ${target}"
          """
          echo "→ App URL: ${params.DEPLOY_ENV == 'DEV' ? APP_URL_DEV : APP_URL_UAT}"
        }
      }
    }

    stage('Data Migration') {
      when { expression { params.DEPLOY_ENV == 'DEV' } }
      steps {
        sh '''
          # ensure UAT data exists
          if [ ! -f "${UAT_DIR}/data.json" ]; then
            printf '{"foo":"bar","source":"UAT-Server"}' > "${UAT_DIR}/data.json"
          fi
          echo "UAT data:"; cat "${UAT_DIR}/data.json"

          # migrate to DEV
          cp -v "${UAT_DIR}/data.json" "${DEV_DIR}/data.json"
          sed -i 's/UAT-Server/DEV-Server/' "${DEV_DIR}/data.json"

          echo "DEV data:"; cat "${DEV_DIR}/data.json"
        '''
      }
    }

    stage('Verify Migration') {
      steps {
        script {
          if (fileExists("${UAT_DIR}/data.json")) {
            echo "✅ UAT data.json exists"
          } else {
            error "❌ UAT data.json missing"
          }
          if (fileExists("${DEV_DIR}/data.json")) {
            echo "✅ DEV data.json exists"
          } else {
            error "❌ DEV data.json missing"
          }
        }
      }
    }
  }

  post {
    success {
      echo """
      ====================================
      ✅ PIPELINE SUCCESSFUL
      Environment: ${params.DEPLOY_ENV}
      App URL: ${params.DEPLOY_ENV == 'DEV' ? APP_URL_DEV : APP_URL_UAT}
      ====================================
      """
    }
    failure {
      echo "❌ PIPELINE FAILED – check the console output for details."
    }
  }
}
