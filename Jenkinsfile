pipeline {
    agent any

    tools {
        maven 'Maven3.6'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
        }

        stage('UnitTest') {
            steps {
                echo 'Skipping unit tests...'
                sh 'mvn test -DskipTests'
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
                }
            }
        }
        
        stage('Deploy') {
            steps {
                echo 'Deploying application...'
            }
        }

    }
}
