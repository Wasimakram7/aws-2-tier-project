pipeline {
  agent any

  environment {
    // replace these or set them in Jenkins global env / credentials
    AWS_ACCOUNT_ID      = '783764594284'
    AWS_DEFAULT_REGION  = 'eu-north-1'
    IMAGE_TAG           = "1.0.${BUILD_NUMBER}"
    FRONTEND_DIR        = 'frontend'
    BACKEND_DIR         = 'backend'
    FRONTEND_ECR_URI    = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/frontend"
    BACKEND_ECR_URI     = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/backend"

    // Sonar values - set SONAR_HOST_URL and SONAR_TOKEN in Jenkins credentials or env
    SONAR_HOST_URL      = "${SONAR_HOST_URL ?: 'http://sonarqube:9000'}"
    SONAR_TOKEN         = "${SONAR_TOKEN ?: ''}"
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
      // use sonar-scanner docker image so you don't need to pre-install sonar-scanner on agent
      steps {
        script {
          // run sonar scanner via docker; mount workspace to /src inside container
          sh """
            echo "Running Sonar Scanner..."
            docker run --rm \
              -v "${env.WORKSPACE}:/src" \
              -e SONAR_HOST_URL=${SONAR_HOST_URL} \
              -e SONAR_TOKEN=${SONAR_TOKEN} \
              sonarsource/sonar-scanner-cli:latest \
              -Dsonar.projectKey=${JOB_NAME}-${BUILD_NUMBER} \
              -Dsonar.sources=/src \
              -Dsonar.host.url=${SONAR_HOST_URL} \
              -Dsonar.login=${SONAR_TOKEN} \
              > ${WORKSPACE}/reports/sonar.txt 2>&1 || true
          """
        }
      }
    }

    stage('Trivy Scan (filesystem)') {
      steps {
        script {
          // run trivy scanning the workspace; output to reports/trivy.txt
          sh """
            echo "Running Trivy filesystem scan..."
            docker run --rm -v "${env.WORKSPACE}:/src" aquasec/trivy:latest fs --no-progress --exit-code 0 --ignore-unfixed /src \
              > ${WORKSPACE}/reports/trivy.txt 2>&1 || true
          """
        }
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
          echo "Combined report stored at reports/combined-report.txt"
        '''
        archiveArtifacts artifacts: 'reports/combined-report.txt, reports/sonar.txt, reports/trivy.txt', allowEmptyArchive: false
      }
    }

    stage('Build & Push Docker Images to ECR') {
      // This stage assumes docker and aws CLI are available on agent (or mounted)
      steps {
        script {
          // login to ECR
          sh """
            echo "Ensure ECR repo exists (create if needed)... (optional step skipped)"
            echo "Logging in to ECR..."
            aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
          """

          // Build frontend (if directory exists)
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

          // Build backend (if directory exists)
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
      // Keep combined report available and print path
      sh 'echo "Reports: $(ls -1 reports || true)" || true'
    }
  }
}
