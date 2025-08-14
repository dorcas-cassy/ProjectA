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
      sh """
        echo '--- DEBUG: Maven version ---'
        mvn -v || true
        echo '--- DEBUG: Top-level tree ---'
        ls -lah || true
      """

      if (fileExists('pom.xml')) {
        sh 'mvn -B -DskipTests clean package'
      } else if (fileExists('build.gradle') || fileExists('build.gradle.kts')) {
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
        sh 'echo "--- DEBUG: listing build dirs ---"; ls -lah target || true; ls -lah build/libs || true'
        error 'WAR_FILE not set — build produced no WAR.'
      }

      sh 'ls -lh "${WAR_FILE}"'
      echo "WAR file found: ${env.WAR_FILE}"
    }
  }
}
