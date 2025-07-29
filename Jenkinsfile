pipeline {
  agent any

  parameters {
    choice(name: 'DEPLOY_ENV',
           choices: ['DEV','UAT'],
           description: 'Where to deploy: DEV=172.25.1.28, UAT=172.25.1.27')
  }

  environment {
    JENKINS_HOME_DIR = "/var/lib/jenkins"
    UAT_IP    = "172.25.1.27"
    DEV_IP    = "172.25.1.28"
    UAT_DIR   = "${JENKINS_HOME_DIR}/${UAT_IP}/data"
    DEV_DIR   = "${JENKINS_HOME_DIR}/${DEV_IP}/data"
    WEB_ROOT  = "${JENKINS_HOME_DIR}/userContent/demo-app"
    APP_URL   = "http://${UAT_IP}:8181/userContent/demo-app/index.html"
  }

  stages {
    stage('Debug') {
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

    stage('Build App') {
      steps {
        script {
          writeFile file: 'index.html', text: """
            <!DOCTYPE html><html><body>
              <h1>CI/CD Demo</h1>
              <p>Env: ${params.DEPLOY_ENV}</p>
              <p>Build #${BUILD_NUMBER}</p>
            </body></html>
          """
        }
      }
    }

    stage('Deploy') {
      steps {
        script {
          def targetDir = (params.DEPLOY_ENV=='DEV') ? DEV_DIR : UAT_DIR
          sh """
            cp index.html "${targetDir}/"
            cp index.html "${WEB_ROOT}/"
            echo "Deployed to ${targetDir}"
          """
          echo "→ Demo app URL: ${APP_URL}"
        }
      }
    }

    stage('Data Migration') {
      when { expression { params.DEPLOY_ENV=='DEV' } }
      steps {
        sh '''
          # Ensure UAT data exists
          if [ ! -f "${UAT_DIR}/data.json" ]; then
            echo '{"foo":"bar","source":"UAT"}' > "${UAT_DIR}/data.json"
          fi
          echo "UAT data:"; cat "${UAT_DIR}/data.json"
          # Migrate to DEV
          cp "${UAT_DIR}/data.json" "${DEV_DIR}/data.json"
          sed -i 's/UAT/DEV/' "${DEV_DIR}/data.json"
          echo "DEV data:"; cat "${DEV_DIR}/data.json"
        '''
      }
    }
  }

  post {
    success {
      echo """
      ▶ Pipeline succeeded.
      ▶ App published at: ${APP_URL}
      ▶ Check data on:
         - UAT: /var/lib/jenkins/${UAT_IP}/data/data.json
         - DEV: /var/lib/jenkins/${DEV_IP}/data/data.json
      """
    }
    failure {
      echo "Pipeline failed – see logs above for errors."
    }
  }
}
