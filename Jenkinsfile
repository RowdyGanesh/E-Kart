pipeline {

    agent any   // Allows the pipeline to run on any available Jenkins agent

    tools {
        maven 'Maven3.6'   // Defines Maven 3.6 as the build tool
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
                    sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=E-Kart \
                        -Dsonar.projectName="E-Kart Application"
                    '''
                    // Performs static code analysis and publishes results to SonarQube
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

        stage('Deploy') {
            steps {
                echo 'Deploying application...'      
                // Placeholder stage for deployment logic
            }
        }

    }
}
