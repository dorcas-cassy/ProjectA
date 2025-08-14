pipeline {
    agent any

    tools {
        maven 'M3'
        jdk 'JDK17'
    }

    environment {
        WAR_FILE = ''
        TOMCAT_USER = 'admin'
        TOMCAT_PASS = 'admin'
        TOMCAT_HOST = 'localhost'
        TOMCAT_PORT = '8080'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/dorcas-cassy/ProjectA.git'
            }
        }

        stage('Build WAR') {
            steps {
                sh 'mvn -B -DskipTests clean package'

                script {
                    // Look for the WAR file
                    WAR_FILE = sh(script: "find target -name '*.war' | head -n 1", returnStdout: true).trim()
                    if (!WAR_FILE) {
                        error "WAR_FILE not set – build produced no WAR"
                    } else {
                        echo "WAR file found: ${WAR_FILE}"
                    }
                }
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                script {
                    if (!WAR_FILE) {
                        error "No WAR file to deploy"
                    }
                    sh """
                        curl -u ${TOMCAT_USER}:${TOMCAT_PASS} \\
                        --upload-file ${WAR_FILE} \\
                        "http://${TOMCAT_HOST}:${TOMCAT_PORT}/manager/text/deploy?path=/myapp&update=true"
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployment successful!"
        }
        failure {
            echo "Build/Deploy failed — check logs."
        }
    }
}
