pipeline {
  agent any

  options {
    timestamps()
  }

  // Use the Maven you added in Manage Jenkins → Tools (you named it "M3")
  tools {
    maven 'M3'
  }

  environment {
    // --- Tomcat info (Jenkins in Docker on Mac -> use host.docker.internal) ---
    TOMCAT_URL = 'http://host.docker.internal:8080'
    APP_NAME   = 'helloworld'     // the context path -> http://localhost:8080/helloworld/
    WAR_FILE   = ''               // will be filled after build
  }

  stages {
    stage('Checkout') {
      steps {
        // If your job is "Pipeline script" (not from SCM), use this:
        git url: 'https://github.com/dorcas-cassy/ProjectA.git', branch: 'main'
        // If your job already uses "Pipeline script from SCM", you can remove the line above.
      }
    }

    stage('Build WAR') {
      steps {
        script {
          // Ensure this is a Maven project
          if (!fileExists('pom.xml')) {
            error "No pom.xml found at workspace root -> cannot build WAR."
          }

          // Build
          sh 'mvn -B -DskipTests clean package'

          // Find the WAR produced by the build
          env.WAR_FILE = sh(
            script: "ls -1 target/*.war 2>/dev/null | head -n 1",
            returnStdout: true
          ).trim()

          if (!env.WAR_FILE) {
            // Give a helpful directory listing if nothing was found
            sh 'echo "No WAR in target/. Contents:" && ls -al target || true'
            error "WAR_FILE not set — build produced no WAR. Check Maven output and packaging."
          }

          sh "echo 'WAR file found: ${env.WAR_FILE}' && ls -lh '${env.WAR_FILE}'"
        }
      }
    }

    stage('Deploy to Tomcat') {
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'tomcat-creds',   // your Jenkins credential ID
            usernameVariable: 'TOMCAT_USER',
            passwordVariable: 'TOMCAT_PASS'
          )
        ]) {
          sh """
            set -euo pipefail
            echo 'Undeploying old app (ignore error if not deployed yet)...'
            curl -fsS -u "$TOMCAT_USER:$TOMCAT_PASS" \\
              "${TOMCAT_URL}/manager/text/undeploy?path=/${APP_NAME}" || true

            echo 'Deploying new WAR...'
            curl -fsS -u "$TOMCAT_USER:$TOMCAT_PASS" \\
              -T "${WAR_FILE}" \\
              "${TOMCAT_URL}/manager/text/deploy?path=/${APP_NAME}&update=true"

            echo '✅ Deploy request sent to Tomcat Manager.'
          """
        }
      }
    }
  }

  post {
    success { echo "✅ Deployed ${env.APP_NAME} to ${env.TOMCAT_URL} (WAR=${env.WAR_FILE})" }
    failure { echo "❌ Build/Deploy failed — check stage logs above." }
  }
}
