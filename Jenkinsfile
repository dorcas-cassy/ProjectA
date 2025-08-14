pipeline {
  agent any
  options {
    timestamps()
    ansiColor('xterm')
  }

  tools {
    maven 'M3'      // the Maven you added in Manage Jenkins > Tools
  }

  environment {
    // --- Repo info ---
    REPO   = 'https://github.com/dorcas-cassy/ProjectA.git'
    BRANCH = 'main'

    // --- Tomcat info ---
    // Jenkins is in Docker; Tomcat is on your Mac -> use host.docker.internal
    TOMCAT_URL = 'http://host.docker.internal:8080'
    APP_NAME   = 'helloworld'     // the context path

    // Filled after build
    WAR_FILE = ''
  }

  stages {
    stage('Checkout') {
      steps {
        git url: env.REPO, branch: env.BRANCH
      }
    }

    stage('Build WAR') {
      steps {
        script {
          if (fileExists('pom.xml')) {
            sh 'mvn -B -DskipTests clean package'
            env.WAR_FILE = sh(script: "ls target/*.war | head -n1", returnStdout: true).trim()
          } else if (fileExists('build.gradle')) {
            sh './gradlew clean war -x test'
            env.WAR_FILE = sh(script: "ls build/libs/*.war | head -n1", returnStdout: true).trim()
          } else {
            error 'No pom.xml or build.gradle found — cannot build WAR'
          }
        }
      }
    }

    stage('Deploy to Tomcat') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'tomcat-creds',  // you already created this
          usernameVariable: 'TOMCAT_USER',
          passwordVariable: 'TOMCAT_PASS'
        )]) {
          sh """
            # undeploy (ignore failure if not deployed yet)
            curl -sS -u "$TOMCAT_USER:$TOMCAT_PASS" \
              "${TOMCAT_URL}/manager/text/undeploy?path=/${APP_NAME}" || true

            # deploy new WAR
            curl -sS -f -u "$TOMCAT_USER:$TOMCAT_PASS" \
              -T "${WAR_FILE}" \
              "${TOMCAT_URL}/manager/text/deploy?path=/${APP_NAME}&update=true"
          """
        }
      }
    }
  }

  post {
    success { echo "✅ Deployed ${env.APP_NAME} to ${env.TOMCAT_URL}" }
    failure { echo "❌ Build/Deploy failed — check stage logs." }
  }
}
