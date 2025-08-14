pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
  }

  tools {
    maven 'M3'          // Manage Jenkins → Tools → Maven installations → Name = M3
  }

  environment {
    // Tomcat info
    TOMCAT_URL = 'http://host.docker.internal:8080'
    MANAGER    = "${env.TOMCAT_URL}/manager/text"
    APP_NAME   = 'helloworld'      // your context path → http://localhost:8080/helloworld/
    WAR_FILE   = ''                 // will be set after build
  }

  stages {
    stage('Checkout') {
      steps {
        // If this Jenkinsfile is in your GitHub repo and the job is “Pipeline script from SCM”,
        // `checkout scm` is perfect. If you paste this into a freestyle Pipeline job, use a Git step instead.
        checkout scm
      }
    }

    stage('Build WAR') {
      steps {
        // Build & list artifacts
        sh '''
          set -euxo pipefail
          mvn -v
          rm -rf target || true
          mvn -B -DskipTests clean package
          ls -lah target || true
        '''
        // Capture the newest WAR path into env.WAR_FILE
        script {
          def found = sh(script: "ls -t target/*.war 2>/dev/null | head -n1 || true",
                         returnStdout: true).trim()
          if (!found) {
            error 'WAR build produced no target/*.war'
          }
          env.WAR_FILE = found
          echo "WAR file selected: ${env.WAR_FILE}"
        }
      }
    }

    stage('Deploy to Tomcat') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'tomcat-creds',   // Jenkins Credentials (admin / your-password)
          usernameVariable: 'TOMCAT_USER',
          passwordVariable: 'TOMCAT_PASS'
        )]) {
          sh '''
            set -euxo pipefail
            # Undeploy old version (ignore if not present)
            curl -sfS -u "$TOMCAT_USER:$TOMCAT_PASS" \
              "${MANAGER}/undeploy?path=/${APP_NAME}" || true

            # Deploy the new WAR
            curl -sfS -u "$TOMCAT_USER:$TOMCAT_PASS" \
              -T "${WAR_FILE}" \
              "${MANAGER}/deploy?path=/${APP_NAME}&update=true"
          '''
        }
      }
    }
  }

  post {
    success {
      echo "✅ Deployed /${env.APP_NAME} to ${env.TOMCAT_URL} (WAR=${env.WAR_FILE})"
    }
    failure {
      echo "❌ Build/Deploy failed — check stage logs above."
    }
  }
}
