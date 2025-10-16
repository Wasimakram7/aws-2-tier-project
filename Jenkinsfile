pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'eu-north-1'
        AWS_ACCOUNT_ID     = '783764594284'
        IMAGE_TAG          = "1.0.${BUILD_NUMBER}"
        SCANNER_HOME       = tool 'sonar-scanner'
        FRONTEND_ECR_URI   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/frontend-repo"
        BACKEND_ECR_URI    = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/backend-repo"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-cred',
                    url: 'https://github.com/Wasimakram7/aws-2-tier-project.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \\
                            -Dsonar.projectName=app \\
                            -Dsonar.projectKey=app
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        stage('OWASP Dependency-Check Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Authenticate with AWS and ECR') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials-id']
                ]) {
                    sh '''
                        echo "Authenticating with AWS..."
                        export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                        export AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION}

                        aws sts get-caller-identity

                        echo "Logging into ECR..."
                        aws ecr get-login-password --region $AWS_DEFAULT_REGION | \
                        docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
                    '''
                }
            }
        }

        stage('Build, Tag & Push Frontend Docker Image to AWS ECR') {
            steps {
                dir('frontend') {
                    sh """
                        echo "Building Frontend Docker image..."
                        docker build -t frontend:latest .

                        echo "Tagging image with ${IMAGE_TAG} and latest..."
                        docker tag frontend:latest ${FRONTEND_ECR_URI}:${IMAGE_TAG}
                        docker tag frontend:latest ${FRONTEND_ECR_URI}:latest

                        echo "Pushing images to ECR..."
                        docker push ${FRONTEND_ECR_URI}:${IMAGE_TAG}
                        docker push ${FRONTEND_ECR_URI}:latest
                        sleep 5
                    """
                }
            }
        }

        stage('Scan Latest Frontend Docker Image using Trivy') {
            steps {
                sh "trivy image ${FRONTEND_ECR_URI}:${IMAGE_TAG}"
            }
        }

        stage('Build, Tag & Push Backend Docker Image to AWS ECR') {
            steps {
                dir('backend') {
                    sh """
                        echo "Building Backend Docker image..."
                        docker build -t backend:latest .

                        echo "Tagging image with ${IMAGE_TAG} and latest..."
                        docker tag backend:latest ${BACKEND_ECR_URI}:${IMAGE_TAG}
                        docker tag backend:latest ${BACKEND_ECR_URI}:latest

                        echo "Pushing images to ECR..."
                        docker push ${BACKEND_ECR_URI}:${IMAGE_TAG}
                        docker push ${BACKEND_ECR_URI}:latest
                        sleep 5
                    """
                }
            }
        }

        stage('Scan Latest Backend Docker Image using Trivy') {
            steps {
                sh "trivy image ${BACKEND_ECR_URI}:${IMAGE_TAG}"
            }
        }

        stage('Clean Workspace for CD Repo') {
            steps {
                cleanWs()
            }
        }

       
            }
        }
    }
}
