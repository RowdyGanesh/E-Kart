pipeline {

    agent any                          // Allows the pipeline to run on any available Jenkins agent

    tools {
        maven 'Maven3.6'               // Defines Maven 3.6 as the build tool configured in Jenkins
    }

    environment {

        NEXUS_URL       = 'http://nexus.rowdyops.click:8081'    // Base URL of Nexus Repository Manager
        SNAPSHOT_REPO   = 'rowdyops_maven-snapshots'            // Target Nexus hosted repository for SNAPSHOT artifacts
        NEXUS_REPO      = 'rowdyops_maven-releases'             // Target Nexus hosted repository for RELEASE artifacts
        APP_NAME        = 'shopping-cart'                       // Logical application name used for logging and traceability
        BUILD_ENV       = 'dev'                                 // Environment identifier (dev / tst / uat / prod)
                                                                // Currently informational only – does not affect build logic

        MAVEN_OPTS      = '-Xmx1024m'                           // JVM memory options for Maven execution
    }
    
    stages {

        stage('Checkout') {
            steps {
                checkout scm                         // Pulls source code from the configured Git repository
            }
        }

        
        stage('Build') {
            steps {
                sh 'mvn clean install -DskipTests'   // Builds and packages the application while skipping tests
            }
        }

        
        stage('UnitTest') {
            steps {
                echo 'Skipping unit tests...'        
                sh 'mvn test -DskipTests'            // Unit tests are skipped for faster pipeline execution
            }
        }

        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {      
                    sh 'mvn sonar:sonar'            // Runs SonarQube analysis
                                                    // Uses sonar-project.properties from repository
                }
            }
        }

        
        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {  
                    waitForQualityGate abortPipeline: true    // Fails the pipeline if SonarQube Quality Gate fails
                }
            }
        }

        
        stage('Publish Artifact to Nexus') {
            steps {

                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                // Injects Nexus credentials securely from Jenkins

                    configFileProvider([configFile(
                        fileId: 'maven-nexus-settings',
                        variable: 'MAVEN_SETTINGS'
                    )]) {
                    // Provides custom Maven settings.xml for Nexus authentication

                        script {

                            // Read project version from pom.xml
                            def version = sh(
                                script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                                returnStdout: true
                            ).trim()

                            // Expose version as environment variable (for logging/post steps)
                            env.APP_VERSION = version

                            // Decide Nexus repository based on version type
                            def targetRepo = version.contains('SNAPSHOT')
                                ? SNAPSHOT_REPO
                                : NEXUS_REPO

                            echo "Deploying ${APP_NAME} version ${APP_VERSION} to ${targetRepo}"

                            sh """
                                mvn deploy -DskipTests -s ${MAVEN_SETTINGS}
                            """
                            // Deploys artifact to appropriate Nexus repository
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ ${APP_NAME} ${APP_VERSION} build, SonarQube analysis, and Nexus deployment completed successfully!"
        }
        failure {
            echo "❌ ${APP_NAME} pipeline failed. Check logs for details."
        }
    }
}
