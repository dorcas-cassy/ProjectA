pipeline {
  agent any

  options {
    timestamps()
  }

  tools {
    maven 'M3'   // Manage Jenkins → Tools (your Maven install name)
  }

  environment {
    // --- Repo info ---
    REPO   = 'https://github.com/dorcas-cassy/ProjectA.git'
    BRANCH = 'main'

    // --- Tomcat info (Tomcat runs on your Mac host) ---
    TOMCAT_URL = 'http://host.docker.internal:8080'
    APP_NAME   = 'helloworld'     // context path => http://localhost:8080/helloworld/
    WAR_FILE   = ''               // will be filled after build
  }

  stages {

    stage('Declarative: Checkout SCM') {
      steps {
        git url: env.REPO, branch: env.BRANCH
      }
    }

    stage('Declarative: Tool Install') {
      steps {
        // no-op stage; the 'tools' block above installs Maven for us
        echo "Using Maven tool M3"
      }
    }

    stage('Checkout') {
      steps {
        // keep a simple checkout stage for the blue ocean view
        echo "Repo: ${env.REPO}  Branch: ${env.BRANCH}"
      }
    }

    stage('Build WAR') {
      steps {
        // clean & build with Maven (fast, no tests)
        sh '''
          set -e
          rm -rf target || true
          mvn -B -DskipTests clean package
        '''

        // discover the WAR file we just built
        script {
          def found = sh(
            script: '''
              set -e
              f=""
              if [ -d target ]; then
                f=$(ls -1 target/*.war 2>/dev/null | head -n 1 || true)
              fi
              if [ -z "$f" ]; then
                # last resort: look up to 3 dirs deep
                f=$(find . -maxdepth 3 -type f -name "*.war" | head -n 1 || true)
              fi
              echo "$f"
            ''',
            returnStdout: true
          ).trim()

          if (!found) {
            error 'WAR_FILE not set — build produced no WAR.'
          }
          env.WAR_FILE = found
          echo "WAR file found: ${env.WAR_FILE}"
        }
      }
    }

    stage('Deploy to Tomcat') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'tomcat-creds',          // Jenkins cred you created
          usernameVariable: 'TOMCAT_USER',
          passwordVariable: 'TOMCAT_PASS'
        )]) {
          sh '''
            set -euo pipefail

            echo "==> Checking Tomcat reachability: ${TOMCAT_URL}"
            curl -sS -I "${TOMCAT_URL}" | sed -n '1p'

            echo "==> Checking Manager /text/list with auth"
            LIST_STATUS=$(curl -sS -o /tmp/list.out -w "%{http_code}" \
              -u "$TOMCAT_USER:$TOMCAT_PASS" \
              "${TOMCAT_URL}/manager/text/list" || true)
            echo "Manager /list HTTP ${LIST_STATUS}"
            if [ "${LIST_STATUS}" != "200" ]; then
              echo "---- Manager response ----"
              cat /tmp/list.out || true
              echo "Hint: user must have role 'manager-script' and Manager app must be installed."
              exit 1
            fi

            echo "==> Undeploy old app (ignore if missing)"
            curl -fsS -u "$TOMCAT_USER:$TOMCAT_PASS" \
              "${TOMCAT_URL}/manager/text/undeploy?path=/${APP_NAME}" || true

            echo "==> Deploying WAR: ${WAR_FILE} -> /${APP_NAME}"
            DEPLOY_STATUS=$(curl -sS -o /tmp/deploy.out -w "%{http_code}" \
              -u "$TOMCAT_USER:$TOMCAT_PASS" \
              -T "${WAR_FILE}" \
              "${TOMCAT_URL}/manager/text/deploy?path=/${APP_NAME}&update=true" || true)
            echo "Deploy HTTP ${DEPLOY_STATUS}"
            echo "---- Deploy response ----"
            cat /tmp/deploy.out || true

            case "${DEPLOY_STATUS}" in
              20*) echo "Deploy OK";;
              *)   echo "Deploy failed (HTTP ${DEPLOY_STATUS})"; exit 1;;
            esac
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
      echo "❌ Build/Deploy failed — check stage logs above for exact HTTP status and message."
    }
  }
}
