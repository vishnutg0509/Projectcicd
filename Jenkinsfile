
pipeline {
    agent any

    tools {
        maven 'Maven3'
    }

    environment {
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-amd64'
        PATH = "/usr/lib/jvm/java-17-openjdk-amd64/bin:${env.PATH}"
        AWS_REGION = 'ap-southeast-2'
        K8S_MASTER_ID = 'i-07e2d2a5f3857c704'
        DOCKER_IMAGE = 'vishnutg0509/projectcicd-pipeline'
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
                sh '/usr/local/bin/aws --version'
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

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh './mvnw -B org.sonarsource.scanner.maven:sonar-maven-plugin:3.11.0.3922:sonar -Dsonar.projectKey=Boardgame -DskipTests'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                    docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy --cache-dir /tmp/trivy-image-cache image \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        --no-progress \
                        ${DOCKER_IMAGE}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                    docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                    docker push ${DOCKER_IMAGE}:latest
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    set -eu

                    PARAM_FILE=$(mktemp)
                    trap 'rm -f "$PARAM_FILE"' EXIT

                    printf '{"commands":["set -e; export KUBECONFIG=/home/ssm-user/.kube/config; kubectl set image deployment/projectcicd projectcicd=vishnutg0509/projectcicd-pipeline:%s -n default && kubectl rollout status deployment/projectcicd --timeout=180s -n default"]}' "$BUILD_NUMBER" > "$PARAM_FILE"

                    COMMAND_ID=$(/usr/local/bin/aws ssm send-command \
                        --instance-ids "$K8S_MASTER_ID" \
                        --document-name AWS-RunShellScript \
                        --parameters "file://${PARAM_FILE}" \
                        --region "$AWS_REGION" \
                        --query 'Command.CommandId' \
                        --output text)

                    echo "SSM deployment command: $COMMAND_ID"

                    /usr/local/bin/aws ssm wait command-executed \
                        --command-id "$COMMAND_ID" \
                        --instance-id "$K8S_MASTER_ID" \
                        --region "$AWS_REGION" || true

                    STATUS=$(/usr/local/bin/aws ssm get-command-invocation \
                        --command-id "$COMMAND_ID" \
                        --instance-id "$K8S_MASTER_ID" \
                        --region "$AWS_REGION" \
                        --query 'Status' \
                        --output text)

                    /usr/local/bin/aws ssm get-command-invocation \
                        --command-id "$COMMAND_ID" \
                        --instance-id "$K8S_MASTER_ID" \
                        --region "$AWS_REGION" \
                        --query '{Output:StandardOutputContent,Error:StandardErrorContent}' \
                        --output json

                    if [ "$STATUS" != "Success" ]; then
                        echo "Kubernetes deployment failed. SSM status: $STATUS"
                        exit 1
                    fi

                    echo "Kubernetes deployment completed successfully."
                '''
            }
        }
    }

    post {
        success {
            echo 'Build, scans, Docker push and Kubernetes deployment completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Check the stage logs.'
        }
    }
}
