pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
  }

  // Use the Maven you configured at Manage Jenkins ➜ Tools (named M3 in your screenshot)
  tools {
    maven 'M3'
  }

  environment {
    // --- Repo info ---
    REPO   = 'https://github.com/dorcas-cassy/ProjectA.git'
    BRANCH = 'main'

    // --- Tomcat info ---
    // Jenkins is in Docker; Tomcat is on your Mac — use host.docker.internal
    TOMCAT_URL = 'http://host.docker.internal:8080'
    APP_NAME   = 'helloworld'   // context path -> http://localhost:8080/helloworld/
    WAR_FILE   = ''             // will be filled after build
  }

  stages {

    stage('Checkout') {
      steps {
        git url: env.REPO, branch: env.BRANCH
      }
    }

    stage('Build WAR') {
      steps {
        // helpful debug
        sh 'mvn -v || true'
        sh 'which mvn || true'

        // clean build and produce WAR
        sh 'rm -rf target'
        sh 'mvn -B -DskipTests clean package'

        // capture the produced WAR path
        script {
          env.WAR_FILE = sh(
            script: "ls -1 target/*.war | head -n 1",
            returnStdout: true
          ).trim()

          if (!env.WAR_FILE?.trim()) {
            error "WAR_FILE not set — build produced no WAR."
          }
        }

        sh "ls -lh '${env.WAR_FILE}'"
        echo "WAR file found: ${env.WAR_FILE}"
      }
    }

    stage('Deploy to Tomcat') {
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'tomcat-creds',   // the ID you created
            usernameVariable: 'TOMCAT_USER',
            passwordVariable: 'TOMCAT_PASS'
          )
        ]) {
          sh '''
            set -euo pipefail
            # Undeploy the old app (ignore error if not deployed yet)
            curl -fsS -u "$TOMCAT_USER:$TOMCAT_PASS" \
              "$TOMCAT_URL/manager/text/undeploy?path=/$APP_NAME" || true

            # Deploy the new WAR
            curl -fsS -u "$TOMCAT_USER:$TOMCAT_PASS" \
              -T "$WAR_FILE" \
              "$TOMCAT_URL/manager/text/deploy?path=/$APP_NAME&update=true"
          '''
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
