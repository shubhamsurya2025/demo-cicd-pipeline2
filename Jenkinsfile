pipeline {
  agent any

  /******************************************************************
   * ----- USER-SELECTABLE ENVIRONMENT ------------------------------
   ******************************************************************/
  parameters {
    choice(name: 'DEPLOY_ENV',
           choices: ['DEV', 'UAT'],
           description: 'Where do you want to deploy?')
  }

  /******************************************************************
   * ----- FOLDERS THAT “ACT” AS SERVERS ----------------------------
   *      (all local – no SSH / credentials required)
   ******************************************************************/
  environment {
    UAT_DIR   = "${JENKINS_HOME}/server-27-uat"   // simulates 1.1.1.1.27
    DEV_DIR   = "${JENKINS_HOME}/server-28-dev"   // simulates 1.1.1.1.28
    WEB_ROOT  = "${JENKINS_HOME}/userContent/demo-app"
    JOB_URL   = "${JENKINS_URL}/job/${JOB_NAME}/${BUILD_NUMBER}"
    APP_URL   = "${JENKINS_URL}/userContent/demo-app/index.html"
  }

  stages {

    /*************************  SET-UP  ********************************/
    stage('Prepare folders') {
      steps {
        sh '''
          mkdir -p "${UAT_DIR}/data" "${DEV_DIR}/data" "${WEB_ROOT}"
          echo "Local server folders ready:"
          ls -ld "${UAT_DIR}" "${DEV_DIR}" "${WEB_ROOT}"
        '''
      }
    }

    /*************************  BUILD  *********************************/
    stage('Build Web App') {
      steps {
        script {
          def html = """
          <!DOCTYPE html><html><head>
            <meta charset='utf-8'><title>Demo – ${params.DEPLOY_ENV}</title>
          </head><body style='font-family:sans-serif;text-align:center;
                               padding-top:40px;background:#222;color:#fff'>
            <h1>CI/CD Demo ✔</h1>
            <h2>Environment : ${params.DEPLOY_ENV}</h2>
            <p>Build #${env.BUILD_NUMBER} – ${new Date()}</p>
          </body></html>
          """
          writeFile file: 'index.html', text: html
        }
        echo 'Web page created.'
      }
    }

    /************************  DEPLOY  *********************************/
    stage('Deploy App') {
      steps {
        script {
          def target = params.DEPLOY_ENV == 'DEV' ? env.DEV_DIR : env.UAT_DIR
          sh """
            cp index.html "${target}/"
            cp index.html "${WEB_ROOT}/"     # publish for browser access
            echo "Deployed index.html to ${target}"
          """
        }
      }
    }

    /******************  DATA MIGRATION (DEV only)  ********************/
    stage('Data Migration UAT → DEV') {
      when { expression { params.DEPLOY_ENV == 'DEV' } }
      steps {
        sh '''
          # create sample data on UAT if it is missing
          if [ ! -f "${UAT_DIR}/data/customers.json" ]; then
            cat > "${UAT_DIR}/data/customers.json" <<EOF
            { "customers":[
                {"id":1,"name":"Alice","tier":"gold"},
                {"id":2,"name":"Bob","tier":"silver"}
              ],
              "source":"UAT-Server-27",
              "created":"$(date -Iseconds)"
            }
EOF
          fi

          # migrate to DEV
          cp  "${UAT_DIR}/data/customers.json" \
              "${DEV_DIR}/data/customers.json"
          sed -i 's/UAT-Server-27/DEV-Server-28/' \
              "${DEV_DIR}/data/customers.json"

          echo "Data migrated: $(wc -c < "${DEV_DIR}/data/customers.json") bytes"
        '''
      }
    }

    /*********************  PUBLISH REPORT  ****************************/
    stage('Publish summary') {
      steps {
        script {
          def md = """
          # CI/CD Demo Report

          * **Build:** #${env.BUILD_NUMBER}
          * **Environment:** ${params.DEPLOY_ENV}
          * **Web URL:** ${env.APP_URL}
          * **Status:** SUCCESS

          ${params.DEPLOY_ENV == 'DEV' ?
             '*Data migration from UAT → DEV completed.*' :
             '*UAT deployment – no data migration performed.*'}
          """
          writeFile file: 'REPORT.md', text: md
          writeFile file: 'APP_URL.txt', text: env.APP_URL
        }
        archiveArtifacts artifacts: 'REPORT.md,APP_URL.txt', fingerprint: true
      }
    }
  }

  /**********************  FINAL MESSAGES  *****************************/
  post {
    success {
      echo """
      ============================================
      ✅  PIPELINE SUCCESSFUL
      🌍  Open your browser at →  ${env.APP_URL}
      ============================================
      """
    }
  }
}
