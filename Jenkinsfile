pipeline {

    agent any   // Allows the pipeline to run on any available Jenkins agent

    tools {
        maven 'Maven3.6'   // Defines Maven 3.6 as the build tool
    }

    environment {
        NEXUS_URL  = 'http://nexus.rowdyops.click:8081'
        NEXUS_REPO = 'rowdyops_maven-releases'     // Target Nexus hosted repository for Maven release artifacts
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

        // ============================================================
        // Nexus Deployment Stage
        // ============================================================
        
        stage('Deploy to Nexus') {
                steps {    // Securely fetch Nexus credentials stored in Jenkins
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]){    // Deploys the Maven artifact to Nexus using Jenkins credentials
                    sh """
                        mvn deploy -DskipTests  
                        -DaltDeploymentRepository=nexus::default::${NEXUS_URL}/repository/${NEXUS_REPO}/
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
