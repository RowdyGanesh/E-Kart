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

        stage('Deploy') {
            steps {
                echo 'Deploying application...'
            }
        }

    }
}
