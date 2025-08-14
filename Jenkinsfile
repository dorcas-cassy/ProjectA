pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
  }

  // Make sure you have a Maven tool named "M3" in Manage Jenkins > Tools
  tools {
    maven 'M3'
  }

  environment {
    // --- Tomcat info (Jenkins-in-Docker -> Tomcat-on-Mac) ---
    TOMCAT_URL = 'http://host.docker.internal:8080'
    APP_NAME   = 'helloworld'   // context path -> http://localhost:8080/helloworld/
    // Will be set during build:
    WAR_FILE   = ''
  }

  stages {

    stage('Checkout') {
      steps {
        // Because this Jenkinsfile is in the repo, SCM config in the job supplies 'scm'
        checkout scm
      }
    }

    stage('Build WAR') {
      steps {
        script {
          // Some quick debug so we see what's going on if something fails
          sh """
            echo '--- DEBUG: Java & Maven ---'
            java -version || true
            mvn  -v      || true
            echo '--- DEBUG: Workspace top-level ---'
            ls -lah || true
          """

          if (fileExists('pom.xml')) {
            // Maven build
            sh 'mvn -B -DskipTests clean package'
          } else if (fileExists('build.gradle') || fileExists('build.gradle.kts')) {
            // Gradle build (wrapper preferred, falls back to system gradle)
            sh '''
              chmod +x gradlew || true
              ./gradlew clean build -x test || gradle clean build -x test
            '''
          } else {
            error 'No pom.xml or Gradle build file found — cannot build WAR.'
          }

          // Robust WAR locator: search a few levels deep, ignore .git
          env.WAR_FILE = sh(
            script: "find . -maxdepth 4 -type f -name '*.war' -not -path './.git/*' | head -n 1",
            returnStdout: true
          ).trim()

          sh 'echo "--- DEBUG: found WAR => ${WAR_FILE}"'

          if (!env.WAR_FILE?.trim()) {
            sh 'echo "--- DEBUG: listing common build dirs ---"; ls -lah target || true; ls -lah build/libs || true'
            error 'WAR_FILE not set — build produced no WAR.'
          }

          sh 'ls -lh "${WAR_FILE}"'
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
          sh '''
            set -euo pipefail
            echo "--- Undeploying old version (if present) ---"
            curl -fsS -u "$TOMCAT_USER:$TOMCAT_PASS" \
              "$TOMCAT_URL/manager/text/undeploy?path=/${APP_NAME}" || true

            echo "--- Deploying ${APP_NAME} from ${WAR_FILE} ---"
            curl -fsS -u "$TOMCAT_USER:$TOMCAT_PASS" \
              -T "$WAR_FILE" \
              "$TOMCAT_URL/manager/text/deploy?path=/${APP_NAME}&update=true"
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
