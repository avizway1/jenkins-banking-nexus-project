/*
 * ============================================================
 * Jenkinsfile — The Bank's Automated Assembly Line
 *
 * Analogy:
 *   Think of this pipeline like a car factory conveyor belt.
 *   At each station (stage), something specific happens.
 *   If ANY station fails, the belt stops — nothing broken ships.
 *
 * Stages:
 *   1. Checkout       → Fetch source code from GitHub (raw ingredients)
 *   2. Build Core     → Compile banking-core (bake the cake)
 *   3. Deploy Core    → Push JAR to Nexus (put in the warehouse)
 *   4. Build API      → Compile banking-api (picks core from Nexus)
 *   5. Test           → Run JUnit tests (quality check)
 *   6. Publish API    → Push final JAR to Nexus (ready for delivery)
 * ============================================================
 */

pipeline {

    agent any

    // ============================================================
    // Tools — Define which Maven & Java Jenkins should use.
    // These names must match what you configured in:
    //   Jenkins → Manage Jenkins → Global Tool Configuration
    // ============================================================
    tools {
        maven 'Maven-3.9'       // Name of your Maven installation in Jenkins
        jdk   'Java-21'         // Name of your JDK installation in Jenkins
    }

    environment {
        // Nexus credentials ID — stored in Jenkins Credentials Manager.
        // Never hardcode passwords! Think of this like a key card
        // stored in the security office — referenced by name only.
        NEXUS_CREDENTIALS_ID = 'nexus-credentials'

        // Nexus server URL
        NEXUS_URL = 'http://13.49.243.146:8081'
    }

    stages {

        // ============================================================
        // STAGE 1: Checkout
        // Analogy: The delivery truck arrives with raw ingredients
        //          (source code) from the GitHub warehouse.
        // ============================================================
        stage('Checkout') {
            steps {
                echo '📦 Stage 1: Checking out source code from GitHub...'
                checkout scm
                echo '✅ Source code checked out successfully.'
                sh 'ls -la'
            }
        }

        // ============================================================
        // STAGE 2: Build Core
        // Analogy: The kitchen bakes the "rule book" (banking-core).
        //          Result: banking-core-1.0-SNAPSHOT.jar in target/
        //
        // Note: We use 'install' (not just 'package') so the JAR
        //       goes into the LOCAL Maven cache (.m2) first.
        //       This lets the next deploy step pick it up correctly.
        // ============================================================
        stage('Build Core') {
            steps {
                echo '🔨 Stage 2: Building banking-core module...'
                sh 'mvn -pl banking-core clean install -DskipTests'
                echo '✅ banking-core built and installed to local .m2 cache.'
            }
        }

        // ============================================================
        // STAGE 3: Deploy Core to Nexus
        // Analogy: The baked "rule book" is shipped to the Nexus
        //          warehouse so all teams can pick it up.
        //
        // mvn deploy → compiles, tests (skipped here), packages, AND
        //              uploads the JAR to the Nexus repo defined in
        //              <distributionManagement> in parent pom.xml.
        //
        // Requires: settings.xml on Jenkins server with Nexus creds.
        // ============================================================
        stage('Deploy Core to Nexus') {
            steps {
                echo '🚚 Stage 3: Deploying banking-core JAR to Nexus...'
                // Use withCredentials to inject Nexus username/password
                // into the Maven settings at runtime (secure approach)
                withCredentials([usernamePassword(
                    credentialsId: "${NEXUS_CREDENTIALS_ID}",
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh """
                        mvn -pl banking-core deploy -DskipTests \
                            -Dusername=${NEXUS_USER} \
                            -Dpassword=${NEXUS_PASS}
                    """
                }
                echo '✅ banking-core-1.0-SNAPSHOT.jar deployed to Nexus!'
                echo "   Check: ${NEXUS_URL}/#browse/browse:banking-snapshots"
            }
        }

        // ============================================================
        // STAGE 4: Build API
        // Analogy: The bank teller counter (banking-api) is now being
        //          set up. It automatically pulls the rule book JAR
        //          from Nexus (not from source!) — proving the
        //          warehouse concept works.
        //
        // Maven resolves banking-core dependency from Nexus
        // because it's listed as a dependency in banking-api/pom.xml.
        // ============================================================
        stage('Build API') {
            steps {
                echo '🏗️ Stage 4: Building banking-api (pulls banking-core from Nexus)...'
                sh 'mvn -pl banking-api clean package -DskipTests'
                echo '✅ banking-api built successfully using banking-core from Nexus.'
            }
        }

        // ============================================================
        // STAGE 5: Test
        // Analogy: Quality control — the supervisor runs through all
        //          test scenarios before the teller goes live.
        //
        // Runs JUnit 5 tests in BankControllerTest.java.
        // If any test fails → pipeline stops → nothing goes to Nexus.
        // ============================================================
        stage('Test') {
            steps {
                echo '🧪 Stage 5: Running unit tests on banking-api...'
                sh 'mvn -pl banking-api test'
                echo '✅ All tests passed!'
            }
            post {
                always {
                    // Publish test results in Jenkins UI
                    junit 'banking-api/target/surefire-reports/*.xml'
                }
                failure {
                    echo '❌ Tests failed! Fix before deploying to Nexus.'
                }
            }
        }

        // ============================================================
        // STAGE 6: Publish API Artifact to Nexus
        // Analogy: The finished product (banking-api JAR) is now
        //          boxed, labeled with version 1.0-SNAPSHOT, and
        //          sent to the Nexus warehouse — ready for deployment.
        // ============================================================
        stage('Publish API to Nexus') {
            steps {
                echo '📤 Stage 6: Publishing banking-api artifact to Nexus...'
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
                echo '✅ banking-api-1.0-SNAPSHOT.jar published to Nexus!'
                echo "   Check: ${NEXUS_URL}/#browse/browse:banking-snapshots"
            }
        }
    }

    // ============================================================
    // Post-build actions — Run regardless of success/failure
    // Like the factory floor supervisor writing the shift report.
    // ============================================================
    post {
        success {
            echo '''
            🎉 ==========================================
               Pipeline completed SUCCESSFULLY!
               Flow: GitHub → Build → Nexus → Test → Nexus
               Both JARs are now in the Nexus warehouse.
            ==========================================
            '''
        }
        failure {
            echo '''
            ❌ ==========================================
               Pipeline FAILED! Check the logs above.
               Nothing was deployed — quality gate held.
            ==========================================
            '''
        }
        always {
            echo '🧹 Cleaning workspace...'
            cleanWs()
        }
    }
}
