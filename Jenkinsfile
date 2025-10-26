pipeline {
  agent any

  environment {
    AWS_ACCOUNT_ID      = '78'
    AWS_DEFAULT_REGION  = 'eu-north-1'
    IMAGE_TAG           = "1.0.${BUILD_NUMBER}"
    FRONTEND_DIR        = 'frontend'
    BACKEND_DIR         = 'backend'
    FRONTEND_ECR_URI    = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/frontend"
    BACKEND_ECR_URI     = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/backend"
  }

  options {
    buildDiscarder(logRotator(daysToKeepStr: '14', numToKeepStr: '50'))
    timestamps()
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'mkdir -p reports'
      }
    }

    stage('SonarQube Scan') {
      steps {
        // Use Jenkins Credentials (create a "Secret text" / string credential and set id below)
        withCredentials([string(credentialsId: 'sonar-token-id', variable: 'SONAR_TOKEN')]) {
          sh """
            echo "Running Sonar Scanner (containerized)..."
            docker run --rm \
              -v "${WORKSPACE}:/src" \
              -e SONAR_HOST_URL=${SONAR_HOST_URL:-http://sonarqube:9000} \
              -e SONAR_TOKEN=${SONAR_TOKEN} \
              sonarsource/sonar-scanner-cli:latest \
              -Dsonar.projectKey=${JOB_NAME}-${BUILD_NUMBER} \
              -Dsonar.sources=/src \
              -Dsonar.host.url=${SONAR_HOST_URL:-http://sonarqube:9000} \
              -Dsonar.login=${SONAR_TOKEN} \
              > ${WORKSPACE}/reports/sonar.txt 2>&1 || true
          """
        }
      }
    }

    stage('Trivy Scan (filesystem)') {
      steps {
        sh """
          echo "Running Trivy filesystem scan..."
          docker run --rm -v "${WORKSPACE}:/src" aquasec/trivy:latest fs --no-progress --exit-code 0 --ignore-unfixed /src \
            > ${WORKSPACE}/reports/trivy.txt 2>&1 || true
        """
      }
    }

    stage('Combine Reports') {
      steps {
        sh '''
          echo "Combining Sonar and Trivy reports..."
          {
            echo "==== SonarQube Scan (sonar.txt) ===="
            cat reports/sonar.txt || echo "(no sonar output)"
            echo ""
            echo "==== Trivy Scan (trivy.txt) ===="
            cat reports/trivy.txt || echo "(no trivy output)"
          } > reports/combined-report.txt
        '''
        archiveArtifacts artifacts: 'reports/combined-report.txt, reports/sonar.txt, reports/trivy.txt', allowEmptyArchive: false
      }
    }

    stage('Build & Push Docker Images to ECR') {
      steps {
        script {
          sh """
            echo "Logging in to ECR..."
            aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
          """

          sh """
            if [ -d "${FRONTEND_DIR}" ]; then
              echo "Building frontend image..."
              docker build -t frontend:${IMAGE_TAG} ${FRONTEND_DIR}
              docker tag frontend:${IMAGE_TAG} ${FRONTEND_ECR_URI}:${IMAGE_TAG}
              docker push ${FRONTEND_ECR_URI}:${IMAGE_TAG}
            else
              echo "No frontend dir (${FRONTEND_DIR}) found, skipping frontend build."
            fi
          """

          sh """
            if [ -d "${BACKEND_DIR}" ]; then
              echo "Building backend image..."
              docker build -t backend:${IMAGE_TAG} ${BACKEND_DIR}
              docker tag backend:${IMAGE_TAG} ${BACKEND_ECR_URI}:${IMAGE_TAG}
              docker push ${BACKEND_ECR_URI}:${IMAGE_TAG}
            else
              echo "No backend dir (${BACKEND_DIR}) found, skipping backend build."
            fi
          """
        }
      }
    }
  }

  post {
    success {
      echo "Pipeline finished successfully. Combined report archived at reports/combined-report.txt"
    }
    failure {
      echo "Pipeline failed — check reports/combined-report.txt and console output."
    }
    always {
      sh 'echo "Reports: $(ls -1 reports || true)" || true'
    }
  }
}
