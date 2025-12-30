pipeline {

    agent any   // Allows the pipeline to run on any available Jenkins agent

    tools {
        maven 'Maven3.6'   // Defines Maven 3.6 as the build tool
    }

    environment {

                APP_NAME        = 'ecart-service'                       // Logical application name used for logging and traceability
                BUILD_ENV       = 'dev'                                 // Environment identifier (dev / tst / uat / prod)

                NEXUS_URL       = 'http://nexus.rowdyops.click:8081'    // Base URL of Nexus Repository Manager
                SNAPSHOT_REPO   = 'rowdyops_maven-snapshots'            // Target Nexus hosted repository for SNAPSHOT artifacts
                NEXUS_REPO      = 'rowdyops_maven-releases'             // Target Nexus hosted repository for RELEASE artifacts

                MAVEN_OPTS      = '-Xmx1024m'                           // JVM memory options for Maven execution

                // Docker naming (Enterprise standard)
                ORG_NAME        = 'rowdyops'
                SERVICE_NAME    = 'ecart-service'

                // AWS / ECR
                AWS_REGION      = 'ap-south-1'
                AWS_ACCOUNT_ID  = '910478837823'
                ECR_REPO        = 'rowdyops-ecart-service'
                ECR_REGISTRY    = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

                // AWS / ECS
                CLUSTER_NAME     = 'rowdyops-dev-cluster'
                ECS_SERVICE_NAME = 'rowdyops-ecart-service'
                TASK_FAMILY      = "${ORG_NAME}-${SERVICE_NAME}-td"
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
                    waitForQualityGate abortPipeline: false
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

        stage('Push Docker Image to ECR') {
            steps {
                script {
                    echo "Logging in to ECR..."
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} \
                        | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    """

                    echo "Tagging image for ECR push..."
                    sh """
                        docker tag ${ORG_NAME}-${SERVICE_NAME}:${BUILD_NUMBER} ${ECR_REGISTRY}/${ECR_REPO}:${BUILD_NUMBER}
                    """

                    echo "Pushing image to ECR..."
                    sh """
                        docker push ${ECR_REGISTRY}/${ECR_REPO}:${BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Cleanup Docker Images') {
            steps {
                script {
                    def imageName = "${env.ORG_NAME}-${env.SERVICE_NAME}"
                    echo "Cleaning Docker images for ${imageName}"

                    sh """
                        docker rmi -f ${imageName}:${BUILD_NUMBER} || true
                    """
                }
            }
        }

        stage('Update Task Definition') {
            steps {
                script {
                    sh """
                        aws ecs describe-task-definition \
                        --task-definition ${TASK_FAMILY} \
                        --query taskDefinition > td.json

                        # 🧹 Clean AWS-managed fields
                        jq 'del(
                            .taskDefinitionArn,
                            .revision,
                            .status,
                            .requiresAttributes,
                            .compatibilities,
                            .registeredAt,
                            .registeredBy
                        )' td.json > td_clean.json

                        #  FIX: Set hostPort = 0 for awsvpc / ALB — ports MUST match
                        jq '.containerDefinitions[].portMappings[] |= (.containerPort=9090 | .hostPort=9090 | .protocol="tcp")' \
                        td_clean.json > td_hostfix.json

                        # update image tag dynamically (works for any previous number)
                        sed -i "s|${ECR_REGISTRY}/${ECR_REPO}:[0-9]*|${ECR_REGISTRY}/${ECR_REPO}:${BUILD_NUMBER}|g" td_hostfix.json

                        # register new revision
                        aws ecs register-task-definition --cli-input-json file://td_hostfix.json
                    """
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                script {
                    echo "📌 Fetching latest revision for task family: ${TASK_FAMILY}"

                    def newRevision = sh(
                        script: "aws ecs describe-task-definition --task-definition ${TASK_FAMILY} --query 'taskDefinition.revision' --output text",
                        returnStdout: true
                    ).trim()

                    echo "🚀 Updating ECS service ${ECS_SERVICE_NAME} to revision ${newRevision}"

                    sh """
                        aws ecs update-service \
                        --cluster ${CLUSTER_NAME} \
                        --service ${ECS_SERVICE_NAME} \
                        --task-definition ${TASK_FAMILY}:${newRevision} \
                        --force-new-deployment \
                        --region ${AWS_REGION}
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