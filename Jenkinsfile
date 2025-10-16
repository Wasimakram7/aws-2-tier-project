pipeline {
  agent any

  environment {
    AWS_DEFAULT_REGION = 'eu-north-1'
    AWS_ACCOUNT_ID     = '783764594284'
    IMAGE_TAG          = "1.0.${BUILD_NUMBER}"
    SCANNER_HOME       = tool 'sonar-scanner' // ensure this Tool exists in Jenkins configuration
    FRONTEND_ECR_URI   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/frontend"
    BACKEND_ECR_URI    = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/backend"
  }

  options {
    // keep build logs for a bit
    buildDiscarder(logRotator(daysToKeepStr: '14', numToKeepStr: '50'))
    timestamps()
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        echo "Checked out ${env.BRANCH_NAME ?: 'unknown branch'}"
      }
    }

    stage('Unit Tests - Backend') {
      agent { label 'linux && docker' }
      steps {
        dir('backend') {
          sh '''
            echo "Running backend unit tests..."
            if [ -f build.gradle ]; then
              ./gradlew clean test
            elif [ -f package.json ]; then
              npm ci && npm test
            else
              echo "No known test runner found; skipping"
            fi
          '''
        }
      }
    }

    stage('Build & Push Frontend') {
      agent {
        docker {
          image 'docker:24.0.0'
          args  '-v /var/run/docker.sock:/var/run/docker.sock -v $HOME/.docker:/root/.docker'
        }
      }
      steps {
        dir('frontend') {
          sh '''
            echo "Building frontend Docker image..."
            docker build -t frontend:${IMAGE_TAG} .
            docker tag frontend:${IMAGE_TAG} ${FRONTEND_ECR_URI}:${IMAGE_TAG}
            echo "ECR login..."
            aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
            docker push ${FRONTEND_ECR_URI}:${IMAGE_TAG}
          '''
        }
      }
    }

    stage('Build & Push Backend') {
      agent {
        docker {
          image 'docker:24.0.0'
          args  '-v /var/run/docker.sock:/var/run/docker.sock -v $HOME/.docker:/root/.docker'
        }
      }
      steps {
        dir('backend') {
          sh '''
            echo "Building backend..."
            if [ -f Dockerfile ]; then
              docker build -t backend:${IMAGE_TAG} .
              docker tag backend:${IMAGE_TAG} ${BACKEND_ECR_URI}:${IMAGE_TAG}
              aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
              docker push ${BACKEND_ECR_URI}:${IMAGE_TAG}
            else
              echo "No Dockerfile in backend; run your build steps here (e.g. go build / mvn package)"
            fi
          '''
        }
      }
    }

    stage('Static Analysis / SonarQube') {
      agent { label 'linux' }
      steps {
        withEnv(["PATH+SONAR=${SCANNER_HOME}/bin:${env.PATH}"]) {
          dir('.') {
            sh '''
              echo "Running Sonar Scanner..."
              sonar-scanner \
                -Dsonar.projectKey=myproject \
                -Dsonar.sources=. \
                -Dsonar.host.url=${SONAR_HOST_URL:-http://sonarqube:9000} \
                -Dsonar.login=${SONAR_TOKEN}
            '''
          }
        }
      }
    }

    // OWASP Dependency-Check stage is present but disabled so parser won't choke and it's easy to re-enable.
    stage('OWASP Dependency-Check (disabled)') {
      when {
        expression { return false } // change to 'return true' or remove this when-block to enable
      }
      agent { label 'linux' }
      steps {
        dir('.') {
          sh '''
            echo "OWASP Dependency-Check would run here (stage currently disabled)."
            # Example:
            # dependency-check --project "myproject" --scan . --out reports/dependency-check-report.html
          '''
        }
      }
      post {
        always {
          archiveArtifacts artifacts: 'reports/dependency-check-report.*', allowEmptyArchive: true
        }
      }
    }

    stage('Notify / Deploy') {
      agent { label 'linux' }
      steps {
        echo "Deployment placeholder - implement ECS/K8s/Terraform deploy here."
      }
    }
  }

  post {
    success { echo "Pipeline succeeded - ${env.BUILD_URL}" }
    unstable { echo "Pipeline unstable - check warnings" }
    failure { echo "Pipeline failed - ${env.BUILD_URL}" }
    always {
      // optional cleanup
      sh '''
        echo "Cleanup (non-fatal)..."
        # docker image prune -f || true
      '''
    }
  }
}
