pipeline {
  agent any

  options {
    timestamps()
  }

  tools {
    maven 'M3'              // Manage Jenkins » Tools (your Maven install name)
  }

  environment {
    REPO       = 'https://github.com/dorcas-cassy/ProjectA.git'
    BRANCH     = 'main'

    TOMCAT_URL = 'http://host.docker.internal:8080'   // Tomcat on host Mac
    APP_NAME   = 'helloworld'                          // context path
    WAR_FILE   = ''                                    // filled after build
  }

  stages {

    stage('Checkout') {
      steps {
        git url: env.REPO, branch: env.BRANCH
      }
    }

    stage('Build WAR') {
      steps {
        // clean & build with Maven
        sh '''
          rm -rf target
          mvn -B -DskipTests clean package
        '''

        script {
          // Prefer the WAR Maven puts in target/, else look elsewhere (last resort)
          def found = sh(
            script: '''
              set -e
              if [ -d target ]; then
                f=$(ls -1 target/*.war 2>/dev/null | head -n 1 || true)
              fi
              if [ -z "$f" ]; then
                f=$(find . -maxdepth 3 -type f -name "*.war" | head -n 1 || true)
              fi
              echo "$f"
            ''',
            returnStdout: true
          ).trim()

          if (!found) {
            error 'WAR_FILE not set — build produced no WAR.'
          }
          env.WAR_FILE = found
          echo "WAR file found: ${env.WAR_FILE}"
        }
      }
    }

    stage('Deploy to Tomcat') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'tomcat-creds',
          usernameVariable: 'TOMCAT_USER',
          passwordVariable: 'TOMCAT_PASS'
        )]) {
          sh """
            set -euo pipefail

            # Undeploy old version (ignore if not present)
            curl -fsS -u "$TOMCAT_USER:$TOMCAT_PASS" \
              "${TOMCAT_URL}/manager/text/undeploy?path=/${APP_NAME}" || true

            # Deploy new WAR
            curl -fsS -u "$TOMCAT_USER:$TOMCAT_PASS" \
              -T "${WAR_FILE}" \
              "${TOMCAT_URL}/manager/text/deploy?path=/${APP_NAME}&update=true"
          """
        }
      }
    }
  }

  post {
    success {
      echo "✅ Deployed ${env.APP_NAME} to ${env.TOMCAT_URL} (WAR=${env.WAR_FILE})"
    }
    failure {
      echo "❌ Build/Deploy failed — check stage logs."
    }
  }
}
