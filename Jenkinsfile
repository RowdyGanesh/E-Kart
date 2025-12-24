pipeline {

    agent any   // Allows the pipeline to run on any available Jenkins agent

    tools {
        maven 'Maven3.6'   // Defines Maven 3.6 as the build tool
    }

    environment {
                NEXUS_URL       = 'http://nexus.rowdyops.click:8081'    // Base URL of Nexus Repository Manager
                SNAPSHOT_REPO   = 'rowdyops_maven-snapshots'            // Target Nexus hosted repository for SNAPSHOT artifacts
                NEXUS_REPO      = 'rowdyops_maven-releases'             // Target Nexus hosted repository for RELEASE artifacts
                APP_NAME        = 'shopping-cart'                       // Logical application name used for logging and traceability
                BUILD_ENV       = 'dev'                                 // Environment identifier (dev / tst / uat / prod)

                MAVEN_OPTS      = '-Xmx1024m'                           // JVM memory options for Maven execution

                // Docker naming (Enterprise standard)
                ORG_NAME        = 'rowdyops'
                SERVICE_NAME    = 'ecart-service'
    }
    
    stages {

        stage('Checkout') {
            steps {
                checkout scm   // Pulls source code from the configured Git repository
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean install -DskipTests'   
                // Builds and packages the application while skipping tests
            }
        }

        stage('UnitTest') {
            steps {
                echo 'Skipping unit tests...'        
                sh 'mvn test -DskipTests'            
                // Unit tests are skipped for faster pipeline execution
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {      
                    sh 'mvn sonar:sonar'
                    // Uses sonar-project.properties from repo
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {  
                    waitForQualityGate abortPipeline: true
                    // Fails the pipeline if SonarQube quality gate conditions are not met
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
                configFileProvider([configFile(
                    fileId: 'maven-nexus-settings',
                    variable: 'MAVEN_SETTINGS'
                )]) {
                        script {

                            // Read project version from pom.xml
                            def version = sh(
                                script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                                returnStdout: true
                            ).trim()

                            // Decide repository based on version
                            def targetRepo = version.contains('SNAPSHOT')
                                ? SNAPSHOT_REPO
                                : NEXUS_REPO

                            echo "Deploying version ${version} to ${targetRepo}"

                            sh """
                                mvn deploy -DskipTests -s ${MAVEN_SETTINGS}
                            """
                        }
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    def imageName = "${env.ORG_NAME}-${env.SERVICE_NAME}"
                    def imageTag  = "${BUILD_NUMBER}"

                    echo "Building Docker image ${imageName}:${imageTag}"

                    sh """
                        docker build \
                        -t ${imageName}:${imageTag} \
                        -f docker/Dockerfile .
                    """
                }
            }
        }
    }

    post {
        success {
            echo '✅ Build, SonarQube analysis, and Nexus deployment completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed. Check logs for details.'
        }
            
    }
}
