pipeline {
  agent any
  options { timestamps() }

  tools {
    // Change this if your Maven tool name is different
    maven 'M3'
  }

  environment {
    // Tomcat info
    TOMCAT_URL = 'http://host.docker.internal:8080'
    APP_NAME   = 'helloworld'

    // Will be set during build
    WAR_FILE   = ''
  }

  stages {
    stage('Checkout') {
      steps {
        // Uses the same SCM config as the job
        checkout scm
      }
    }

    stage('Build WAR') {
      steps {
        script {
          if (fileExists('pom.xml')) {
            sh 'mvn -B -DskipTests clean package'
            env.WAR_FILE = sh(
              script: "ls -1 target/*.war | head -n 1",
              returnStdout: true
            ).trim()
          } else if (fileExists('build.gradle') || fileExists('build.gradle.kts')) {
            // Prefer wrapper; fall back to system gradle if needed
            sh 'chmod +x gradlew || true'
            sh './gradlew clean build -x test || gradle clean build -x test'
            env.WAR_FILE = sh(
              script: "ls -1 build/libs/*.war | head -n 1",
              returnStdout: true
            ).trim()
          } else {
            error 'No pom.xml or Gradle build file found — cannot build WAR.'
          }

          if (!env.WAR_FILE?.trim()) {
            error 'WAR_FILE not set — build produced no WAR.'
          }
          sh "ls -lh '${env.WAR_FILE}'"
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
      echo "✅ Deployed /${env.APP_NAME} to ${env.TOMCAT_URL} (WAR=${env.WAR_FILE})"
    }
    failure {
      echo "❌ Build/Deploy failed — check stage logs."
    }
  }
}
