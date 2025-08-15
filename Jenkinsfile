  pipeline {
    agent any

    environment {
        TOMCAT_URL = 'http://host.docker.internal:8080'
        CREDENTIALS_ID = 'tomcat-creds'
        WAR_FILE = 'target/myapp.war'
    }

    stages {
        stage('Checkout SCM') {
            steps {
                git branch: 'main', url: 'https://github.com/yourusername/yourrepo.git'
            }
        }

        stage('Build WAR') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                withCredentials([usernamePassword(credentialsId: CREDENTIALS_ID, usernameVariable: 'TOMCAT_USER', passwordVariable: 'TOMCAT_PASS')]) {
                    sh """
                        curl -v -u $TOMCAT_USER:$TOMCAT_PASS \
                        --upload-file ${WAR_FILE} \
                        ${TOMCAT_URL}/manager/text/deploy?path=/myapp&update=true
                    """
                }
            }
        }
    }
}
