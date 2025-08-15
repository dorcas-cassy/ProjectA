pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
  }

  // Uses the Maven you configured as "M3" in Manage Jenkins › Tools
  tools {
    maven 'M3'
  }

  // Handy overrides you can pass at build time, but with safe defaults
  parameters {
    string(name: 'REPO',                defaultValue: 'https://github.com/dorcas-cassy/ProjectA.git', description: 'Git repo')
    string(name: 'BRANCH',              defaultValue: 'main',                                          description: 'Git branch')
    string(name: 'TOMCAT_URL_OVERRIDE', defaultValue: '',                                              description: 'Use this if host.docker.internal doesn’t work (e.g., http://192.168.x.x:8080)')
  }

  environment {
    APP_NAME = 'helloworld'        // Tomcat context path
    WAR_FILE = ''                  // will be set after build
  }

  stages {
    stage('Prep') {
      steps { deleteDir() }        // start from a clean workspace
    }

    stage('Checkout') {
      steps {
        git url: params.REPO, branch: params.BRANCH
      }
    }

    stage('Build WAR') {
      steps {
        sh '''
          set -e
          rm -rf target
          mvn -B -DskipTests clean package
        '''
        script {
          def found = sh(
            script: 'ls -1 target/*.war 2>/dev/null | head -n 1 || true',
            returnStdout: true
          ).trim()
          if (!found) {
            error 'WAR_FILE not set — build produced no WAR.'
          }
          env.WAR_FILE = found
          echo "WAR file: ${env.WAR_FILE}"
        }
      }
    }

    stage('Deploy to Tomcat') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'tomcat-creds',     // <-- must match your Jenkins credential ID
          usernameVariable: 'TOMCAT_USER',
          passwordVariable: 'TOMCAT_PASS'
        )]) {
          script {
            // 1) Work out the Tomcat URL Jenkins can reach
            def tryUrls = []
            if (params.TOMCAT_URL_OVERRIDE?.trim()) {
              tryUrls << params.TOMCAT_URL_OVERRIDE.trim()
            }
            tryUrls << 'http://host.docker.internal:8080'   // Jenkins-in-Docker → host macOS
            tryUrls << 'http://localhost:8080'              // Jenkins on host

            def good = ''
            for (u in tryUrls) {
              echo "Probing Tomcat manager at ${u}/manager/text/list"
              def rc = sh(
                script: """curl -sS -u "${TOMCAT_USER}:${TOMCAT_PASS}" "${u}/manager/text/list" >/dev/null 2>&1 || true""",
                returnStatus: true
              )
              if (rc == 0) { good = u; break }
            }
            if (!good) {
              error """Could not reach Tomcat manager.
Tried: ${tryUrls.join(', ')}
Tips: 
 - Make sure Tomcat is running and the Manager app is installed.
 - Confirm admin user has roles: manager-gui, manager-script (in tomcat-users.xml).
 - If Jenkins is in Docker, use TOMCAT_URL_OVERRIDE with your Mac’s LAN IP, e.g. http://192.168.x.x:8080"""
            }
            env.TOMCAT_URL = good
            echo "Using TOMCAT_URL=${env.TOMCAT_URL}"

            // 2) Undeploy (ignore if missing) and deploy
            sh """
              set -euo pipefail

              echo "Undeploy existing /${APP_NAME} (ignore errors if not deployed)…"
              curl -fsS -u "${TOMCAT_USER}:${TOMCAT_PASS}" \\
                   "${TOMCAT_URL}/manager/text/undeploy?path=/${APP_NAME}" || true

              echo "Deploying ${WAR_FILE} to /${APP_NAME}…"
              curl -fsS -u "${TOMCAT_USER}:${TOMCAT_PASS}" \\
                   -T "${WAR_FILE}" \\
                   "${TOMCAT_URL}/manager/text/deploy?path=/${APP_NAME}&update=true"

              echo "Deployment done. Verifying list…"
              curl -fsS -u "${TOMCAT_USER}:${TOMCAT_PASS}" "${TOMCAT_URL}/manager/text/list" | sed -n '1,10p'
            """
          }
        }
      }
    }
  }

  post {
    success {
      echo "✅ Deployed ${env.APP_NAME} to ${env.TOMCAT_URL} (WAR=${env.WAR_FILE})"
    }
    failure {
      echo "❌ Build/Deploy failed — open the failed stage logs for the exact reason."
    }
  }
}
