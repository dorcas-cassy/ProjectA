pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
  }

  tools {
    // Manage Jenkins → Tools → Maven installations → Name: M3
    maven 'M3'
  }

  environment {
    // --- Repo info ---
    REPO   = 'https://github.com/dorcas-cassy/ProjectA.git'
    BRANCH = 'main'

    // --- Tomcat info (Jenkins in Docker → Tomcat on Mac) ---
    TOMCAT_URL = 'http://host.docker.internal:8080'
    APP_NAME   = 'helloworld'   // context path -> http://localhost:8080/helloworld/
    WAR_FILE   = ''
  }

  stages {

    stage('Checkout') {
      steps {
        git url: env.REPO, branch: env.BRANCH
      }
    }

    stage('Build WAR') {
      steps {
        // Clean build with Maven
        sh '''
          set -euxo pipefail
          mvn -v
          rm -rf target || true
          mvn -B -DskipTests clean package
        '''

        // Find the generated WAR (be robust about the path)
        script {
          env.WAR_FILE = sh(
            script: '''
              # Prefer target/*.war at repo root
              WAR=$(ls -1 target/*.war 2>/dev/null | head -n 1 || true)
              if [ -z "$WAR" ]; then
                # Fallback: look up to 3 levels deep but still prefer /target/
                WAR=$(find . -maxdepth 3 -type f -name "*.war" | grep "/target/" | head -n 1 || true)
              fi
              echo "$WAR"
            ''',
            returnStdout: true
          ).trim()

          if (!env.WAR_FILE) {
            error "WAR_FILE not set — build produced no WAR. Check that <packaging>war</packaging> is in pom.xml."
          } else {
            echo "WAR file found: ${env.WAR_FILE}"
            sh "ls -lh ${env.WAR_FILE}"
          }
        }
      }
    }

    stage('Deploy to Tomcat') {
      steps {
        withCredentials([
          usernamePassword(credentialsId: 'tomcat-creds',
                           usernameVariable: 'TOMCAT_USER',
                           passwordVariable: 'TOMCAT_PASS')
        ]) {
          sh '''
            set -euo pipefail
            echo "Undeploying old /${APP_NAME} (ignore error if not deployed yet)..."
            curl -sS -u "${TOMCAT_USER}:${TOMCAT_PASS}" \
                 "${TOMCAT_URL}/manager/text/undeploy?path=/${APP_NAME}" || true

            echo "Deploying ${WAR_FILE} to ${TOMCAT_URL} at /${APP_NAME} ..."
            curl -sS -u "${TOMCAT_USER}:${TOMCAT_PASS}" \
                 --upload-file "${WAR_FILE}" \
                 "${TOMCAT_URL}/manager/text/deploy?path=/${APP_NAME}&update=true"
          '''
        }
      }
    }
  }

  post {
    success {
      echo "✅ Deployed /${env.APP_NAME} to ${env.TOMCAT_URL}"
    }
    failure {
      echo "❌ Build/Deploy failed — check stage logs above."
    }
  }
}
