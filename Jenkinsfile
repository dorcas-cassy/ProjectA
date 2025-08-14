pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
  }

  tools {
    maven 'M3'   // Manage Jenkins -> Tools -> Maven installations -> Name = M3
  }

  environment {
    // --- Repo info (informational) ---
    REPO   = 'https://github.com/dorcas-cassy/ProjectA.git'
    BRANCH = 'main'

    // --- Tomcat info ---
    TOMCAT_URL = 'http://host.docker.internal:8080'
    APP_NAME   = 'helloworld'    // context path -> http://localhost:8080/helloworld/

    // Filled after build
    WAR_FILE = ''
  }

  stages {

    stage('Checkout') {
      steps {
        // If you configured "Pipeline script from SCM", Jenkins will check out automatically.
        // This fallback checkout helps when the job uses inline pipeline script.
        checkout([
          $class: 'GitSCM',
          branches: [[name: "*/${env.BRANCH}"]],
          userRemoteConfigs: [[url: env.REPO]]
        ])
      }
    }

    stage('Build WAR') {
      steps {
        script {
          // Build with Maven (skip tests for speed)
          sh '''
            echo '--- DEBUG: Java / Maven versions ---'
            java -version || true
            mvn -v || true
          '''
          sh 'mvn -B -DskipTests clean package'

          // 1) Standard Maven output directory
          def warCandidate = sh(returnStdout: true,
            script: "ls -1 target/*.war 2>/dev/null | head -n 1"
          ).trim()

          // 2) Fallback – search the workspace just in case
          if (!warCandidate) {
            warCandidate = sh(returnStdout: true,
              script: "find . -maxdepth 4 -type f -name '*.war' -not -path './.git/*' | head -n 1"
            ).trim()
          }

          env.WAR_FILE = warCandidate
          sh 'echo "--- DEBUG: WAR_FILE => ${WAR_FILE}"'

          if (!env.WAR_FILE?.trim()) {
            sh 'echo "--- DEBUG: listing target & build/libs ---"; ls -lah target || true; ls -lah build/libs || true'
            error 'WAR_FILE not set — build produced no WAR.'
          }

          sh 'ls -lh "${WAR_FILE}"'
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
          script {
            // (Re)deploy using Tomcat Manager text API
            sh """
              set -euo pipefail

              echo '--- Undeploying old ${APP_NAME} (ignore error if not present) ---'
              curl -sS -u "${TOMCAT_USER}:${TOMCAT_PASS}" \\
                   "${TOMCAT_URL}/manager/text/undeploy?path=/${APP_NAME}" || true

              echo '--- Deploying WAR to /${APP_NAME} ---'
              # retry a couple of times in case Tomcat is busy
              n=0
              until [ \$n -ge 3 ]; do
                curl -sSf -u "${TOMCAT_USER}:${TOMCAT_PASS}" \\
                     -T "${WAR_FILE}" \\
                     "${TOMCAT_URL}/manager/text/deploy?path=/${APP_NAME}&update=true" && break
                n=\$((n+1))
                echo "Deploy attempt \$n failed — retrying in 2s..."; sleep 2
              done
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
      echo "❌ Build/Deploy failed — check stage logs."
    }
  }
}
