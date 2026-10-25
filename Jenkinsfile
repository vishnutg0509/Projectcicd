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

            
            stage('Trivy Image Scan') {
                steps {
                    sh '''
                        trivy \
                          --cache-dir /tmp/trivy-image-cache \
                          image \
                          --severity HIGH,CRITICAL \
                          --exit-code 0 \
                          --no-progress \
                          vishnutg0509/projectcicd-pipeline:${BUILD_NUMBER}
                    '''
                }
        }


        
        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t vishnutg0509/projectcicd-pipeline:${BUILD_NUMBER} .
                    docker tag vishnutg0509/projectcicd-pipeline:${BUILD_NUMBER} \
                      vishnutg0509/projectcicd-pipeline:latest
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 0 \
                      --no-progress \
                      vishnutg0509/projectcicd-pipeline:${BUILD_NUMBER}
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                    docker push vishnutg0509/projectcicd-pipeline:${BUILD_NUMBER}
                    docker push vishnutg0509/projectcicd-pipeline:latest
                '''
            }
        }


        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh './mvnw -B org.sonarsource.scanner.maven:sonar-maven-plugin:3.11.0.3922:sonar -Dsonar.projectKey=Boardgame -DskipTests'
                }
            }
        }
    }

    post {
        success {
            echo 'Build and SonarQube analysis completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Check the stage logs.'
        }
    }
}
