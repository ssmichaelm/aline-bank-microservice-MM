pipeline {
    // Only 3 Jenkins builds are kept
    options {
        buildDiscarder(logRotator(numToKeepStr: '3'))
    }
    agent any
    tools {
        maven 'Apache Maven 3.8.4'
        jdk 'jdk-17.0.2'
    }

    environment {
        REGION = 'us-east-1'
        MICROSERVICE = 'bank'
        AWS_ID = credentials('aws_account_id')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/feature']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/ssmichaelm/aline-bank-microservice-MM.git']]])
                sh 'git submodule init'
                sh 'git submodule update'
            }
        }

        // Continuous Integeration:
        // Compile and test the code
        stage('Compiling') {
            steps {
                sh 'mvn clean compile'
            }
        }
        stage('Testing') {
            steps {
                sh 'mvn clean test'
            }
        }
        // SonarQube
        // Perform "mvn clean verfy" to ensure compiled classes are up-to-date so the Maven Surefire execution from "mvn sonar:sonar" includes the most recent modifications
        stage('SonarQube analysis') {
            steps {
                withSonarQubeEnv('sq1') {
                    sh 'mvn clean verify sonar:sonar'
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        // Produce artifact from the code, ready to be deployed
        stage('Packaging') {
            steps {
                sh 'mvn clean package -Dmaven.test.skip=true'
            }
        }
        stage('Building image') {
            steps {
                sh 'aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com'
                sh 'docker build -t mm-${MICROSERVICE}-microservice:latest . '
            }
        }

        // Deploy the artifact to the server
        stage('Deploying') {
            steps {
                script {
                    env.tag = sh (script: "git rev-parse --short=10 HEAD", returnStdout: true)
                }
                
                sh "docker tag mm-${MICROSERVICE}-microservice:latest ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/mm-${MICROSERVICE}-microservice:latest"
                sh "docker push ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/mm-${MICROSERVICE}-microservice:latest"

                sh "docker tag mm-${MICROSERVICE}-microservice:latest ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/mm-${MICROSERVICE}-microservice:${env.tag}" 
                sh "docker push ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/mm-${MICROSERVICE}-microservice:${env.tag}"
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished'
        }
        cleanup {
            sh "docker rmi ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/mm-${MICROSERVICE}-microservice:latest"
            sh "docker rmi ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/mm-${MICROSERVICE}-microservice:${env.tag}"
        }
    }
}