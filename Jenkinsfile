pipeline {
    agent any

    tools {
        maven 'Maven3'
    }

    environment {
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-amd64'
        PATH = "/usr/lib/jvm/java-17-openjdk-amd64/bin:${env.PATH}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify Tools') {
            steps {
                sh 'java -version'
                sh 'mvn -version'
                sh 'trivy --version'
            }
        }

        stage('Build JAR') {
            steps {
                sh 'chmod +x mvnw'
                sh './mvnw -B clean package -DskipTests'
            }
        }

        stage('Verify JAR') {
            steps {
                sh 'ls -lh target/*.jar'
            }
        }
    }

    post {
        success {
            echo 'Projectcicd build completed successfully.'
        }
        failure {
            echo 'Projectcicd pipeline failed. Check the stage logs.'
        }
    }
}
