pipeline {

    agent any

    tools {
        maven 'maven'
    }

    environment {
        NEXUS_CREDENTIALS_ID  = 'nexus'
        NEXUS_URL             = 'http://13.49.243.146:8081'
        TOMCAT_CREDENTIALS_ID = 'tomcat'                        // Jenkins credential ID for Tomcat
        TOMCAT_URL            = 'http://172.31.30.24:8080'  // Replace with your Tomcat EC2 IP/hostname
        APP_CONTEXT_PATH      = '/banking-app'                  // URL context: http://host:8080/banking-app
    }

    stages {

        stage('Checkout') {
            steps {
                echo '📦 Stage 1: Checking out source code from GitHub...'
                checkout scm
                echo '✅ Source code checked out successfully.'
                sh 'ls -la'
            }
        }

        stage('Build Core') {
            steps {
                echo '🔨 Stage 2: Building banking-core module...'
                sh 'mvn -pl banking-core --also-make clean install -DskipTests'
                echo '✅ banking-core built and installed to local .m2 cache.'
            }
        }

        stage('Deploy Core to Nexus') {
            steps {
                echo '🚚 Stage 3: Deploying parent POM + banking-core JAR to Nexus...'
                withCredentials([usernamePassword(
                    credentialsId: "${NEXUS_CREDENTIALS_ID}",
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh """
                        # Step 3a: Deploy parent POM first (-N = non-recursive, root POM only)
                        # Analogy: Ship the bank's master policy doc to Nexus warehouse first
                        mvn deploy -N -DskipTests \
                            -Dusername=${NEXUS_USER} \
                            -Dpassword=${NEXUS_PASS}

                        # Step 3b: Now deploy banking-core JAR
                        mvn -pl banking-core deploy -DskipTests \
                            -Dusername=${NEXUS_USER} \
                            -Dpassword=${NEXUS_PASS}
                    """
                }
                echo '✅ Parent POM + banking-core-1.0-SNAPSHOT.jar deployed to Nexus!'
            }
        }

        stage('Build API') {
            steps {
                echo '🏗️ Stage 4: Building banking-api (pulls banking-core from Nexus)...'
                sh 'mvn -pl banking-api clean package -DskipTests'
                echo '✅ banking-api built successfully using banking-core from Nexus.'
            }
        }

        stage('Test') {
            steps {
                echo '🧪 Stage 5: Running unit tests on banking-api...'
                sh 'mvn -pl banking-api test'
                echo '✅ All tests passed!'
            }
            post {
                always {
                    junit 'banking-api/target/surefire-reports/*.xml'
                }
                failure {
                    echo '❌ Tests failed! Fix before deploying to Nexus.'
                }
            }
        }

        stage('Publish API to Nexus') {
            steps {
                echo '📤 Stage 6: Publishing banking-api WAR artifact to Nexus...'
                withCredentials([usernamePassword(
                    credentialsId: "${NEXUS_CREDENTIALS_ID}",
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh """
                        mvn -pl banking-api deploy -DskipTests \
                            -Dusername=${NEXUS_USER} \
                            -Dpassword=${NEXUS_PASS}
                    """
                }
                echo '✅ banking-app.war published to Nexus!'
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                echo '🚀 Stage 7: Deploying WAR to Tomcat Server...'
                withCredentials([usernamePassword(
                    credentialsId: "${TOMCAT_CREDENTIALS_ID}",
                    usernameVariable: 'TOMCAT_USER',
                    passwordVariable: 'TOMCAT_PASS'
                )]) {
                    sh """
                        # Undeploy old version first (ignore error if not deployed yet)
                        curl -s -u ${TOMCAT_USER}:${TOMCAT_PASS} \
                            "${TOMCAT_URL}/manager/text/undeploy?path=${APP_CONTEXT_PATH}" || true

                        # Deploy new WAR via Tomcat Manager REST API
                        curl -v -f -u ${TOMCAT_USER}:${TOMCAT_PASS} \
                            -T banking-api/target/banking-app.war \
                            "${TOMCAT_URL}/manager/text/deploy?path=${APP_CONTEXT_PATH}&update=true"
                    """
                }
                echo "✅ WAR deployed! Open: ${TOMCAT_URL}${APP_CONTEXT_PATH}"
            }
        }
    }

    post {
        success {
            echo '🎉 Pipeline completed SUCCESSFULLY! Both JARs are now in the Nexus warehouse.'
        }
        failure {
            echo '❌ Pipeline FAILED! Check the logs above.'
        }
        always {
            echo '🧹 Cleaning workspace...'
            cleanWs()
        }
    }
}
