pipeline {
    agent any

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        ansiColor('xterm')
    }

    environment {
        AWS_DEFAULT_REGION = 'eu-north-1'
        AWS_ACCOUNT_ID     = '783764594284'
        IMAGE_TAG          = "1.0.${BUILD_NUMBER}"
        SCANNER_HOME       = tool 'sonar-scanner'    // ensure this tool is configured in Jenkins Global Tools
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
                        ${SCANNER_HOME}/bin/sonar-scanner \
                            -Dsonar.projectName=app \
                            -Dsonar.projectKey=app
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    // waitForQualityGate returns a map with .status
                    def qg = waitForQualityGate()
                    echo "SonarQube Quality Gate status: ${qg.status}"
                    if (qg.status != 'OK') {
                        // fail the build if quality gate failed
                        error "Pipeline aborted due to SonarQube quality gate: ${qg.status}"
                    }
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
                // 'aws-credentials-id' should be an AWS credential (access key/secret) stored in Jenkins
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials-id']
                ]) {
                    sh '''
                        echo "Authenticating with AWS..."
                        # AWS env vars injected by withCredentials: AWS_ACCESS_KEY_ID & AWS_SECRET_ACCESS_KEY
                        export AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION}

                        aws sts get-caller-identity

                        echo "Logging into ECR..."
                        aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | \
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
                    """
                }
            }
        }

        stage('Scan Frontend Docker Image using Trivy') {
            steps {
                // Ensure trivy is available on the agent or use container step
                sh "trivy image ${FRONTEND_ECR_URI}:${IMAGE_TAG} || true"
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
                    """
                }
            }
        }

        stage('Scan Backend Docker Image using Trivy') {
            steps {
                sh "trivy image ${BACKEND_ECR_URI}:${IMAGE_TAG} || true"
            }
        }

        stage('Prepare for CD / Clean Workspace') {
            steps {
                // do any CD repo checkout or artifact preparation here
                cleanWs()
            }
        }
    }

    post {
        always {
            echo "Pipeline finished - cleaning workspace"
            cleanWs()
        }
        failure {
            echo "Build failed - check console output and reports"
        }
        success {
            echo "Build succeeded - ${IMAGE_TAG} pushed to ECR"
        }
    }
}
