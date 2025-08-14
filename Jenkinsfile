pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
  }

  tools {
    // Use the Maven you defined in Manage Jenkins → Tools (you named it M3)
    maven 'M3'
  }

  environment {
    // --- Repo info ---
    REPO   = 'https://github.com/dorcas-cassy/ProjectA.git'
    BRANCH = 'main'

    // --- Tomcat info ---
    // Jenkins runs in Docker; Tomcat is on your Mac → use host.docker.internal
    TOMCAT_URL = 'http://host.docker.internal:8080'
    APP_NAME   = 'helloworld'        // context path becomes /helloworld

    // Will be filled during build
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
        // Put Maven on PATH for this stage
        withEnv(["PATH=${tool 'M3'}/bin:${env.PATH}"]) {
          sh 'mvn --version || true'
          sh 'rm -rf target || true'

          // Build with Maven if pom.xml exists; else try Gradle (wrapper or system)
          script {
            if (fileExists('pom.xml')) {
              sh 'mvn -B -DskipTests clean package'
            } else if (fileExists('build.gradle') || fileExists('build.gradle.kts')) {
              sh '''
                set -euxo pipefail
                if [ -f ./gradlew ]; then
                  chmod +x ./gradlew
                  ./gradlew clean build -x test
                else
                  gradle clean build -x test
                fi
              '''
            } else {
              error 'No pom.xml or Gradle build file found — cannot build WAR.'
            }
          }

          // Debug: show where the build put artifacts
          sh '''
            echo "== Workspace =="
            pwd
            echo "== Top level =="
            ls -lah || true
            echo "== target/ =="
            ls -lah target || true
            echo "== build/libs/ =="
            ls -lah build/libs || true
            echo "== Find any WAR (depth<=3) =="
            find . -maxdepth 3 -type f -name "*.war" -print || true
          '''

          // Grab the first WAR we find (prefer target/*.war, then build/libs/*.war)
          script {
            def war = sh(
              script: '''
                set -e
                if ls target/*.war >/dev/null 2>&1; then
                  ls -1 target/*.war | head -n 1
                elif ls build/libs/*.war >/dev/null 2>&1; then
                  ls -1 build/libs/*.war | head -n 1
                else
                  # last resort: find anywhere within 3 dirs
                  find . -maxdepth 3 -type f -name "*.war" | head -n 1
                fi
              ''',
              returnStdout: true
            ).trim()

            env.WAR_FILE = war
            if (!env.WAR_FILE) {
              error 'WAR_FILE not set — build produced no WAR.'
            }
            echo "WAR file found: ${env.WAR_FILE}"
          }
        }
      }
    }

    stage('Deploy to Tomcat') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'tomcat-creds',    // Manage Jenkins → Credentials (your admin/****)
          usernameVariable: 'TOMCAT_USER',
          passwordVariable: 'TOMCAT_PASS'
        )]) {
          sh """
            set -euo pipefail
            echo "Undeploying /${APP_NAME} (ignore error if not deployed)..."
            curl -sfS -u "$TOMCAT_USER:$TOMCAT_PASS" \\
              "${TOMCAT_URL}/manager/text/undeploy?path=/${APP_NAME}" || true

            echo "Deploying WAR: ${WAR_FILE} -> /${APP_NAME}"
            curl -sfS -u "$TOMCAT_USER:$TOMCAT_PASS" \\
              -T "${WAR_FILE}" \\
              "${TOMCAT_URL}/manager/text/deploy?path=/${APP_NAME}&update=true"
          """
        }
      }
    }
  }

  post {
    success {
      echo "✅ Deployed ${env.APP_NAME} to ${env.TOMCAT_URL}  (WAR=${env.WAR_FILE})"
    }
    failure {
      echo "❌ Build/Deploy failed — check stage logs."
    }
  }
}
