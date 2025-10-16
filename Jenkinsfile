pipeline {
  agent any

  environment {
    AWS_DEFAULT_REGION = 'eu-north-1'
    AWS_ACCOUNT_ID     = '783764594284'
    IMAGE_TAG          = "1.0.${BUILD_NUMBER}"
    // Adjust SCANNER_HOME to match your Jenkins 'Tool' name for Sonar Scanner
    SCANNER_HOME       = tool 'sonar-scanner'
    // ECR repo URIs - adjust repo names as needed
    FRONTEND_ECR_URI   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/frontend"
    BACKEND_ECR_URI    = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/backend"
  }

  options {
    // keep build logs for a bit
    buildDiscarder(logRotator(daysToKeepStr: '14', numToKeepStr: '50'))
    timestamps()
    ansiColor('xterm')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        echo "Checked out ${env.BRANCH_NAME ?: 'unknown branch'}"
      }
    }

    stage('Unit Tests - Backend') {
      agent {
        label 'linux && docker' // adjust to an agent that can run tests
      }
      steps {
        dir('backend') {
          // Example: run your backend tests; adapt as needed
          sh '''
            echo "Running backend unit tests..."
            # e.g. ./gradlew test or go test ./...
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
      /* Use a docker image with docker CLI so we can build and push images.
         This mounts the host docker socket to allow building with host daemon.
         If you prefer fully isolated DinD, switch to docker:dind and --privileged (extra config needed).
      */
      agent {
        docker {
          image 'docker:24.0.0' // choose appropriate docker CLI image
          args  '-v /var/run/docker.sock:/var/run/docker.sock -v $HOME/.docker:/root/.docker'
        }
      }
      environment {
        // ensure AWS CLI is available in container (install in image or add steps). Alternatively, perform ECR login on host.
      }
      steps {
        dir('frontend') {
          sh '''
            echo "Building frontend Docker image..."
            docker build -t frontend:${IMAGE_TAG} .
            echo "Tagging for ECR..."
            docker tag frontend:${IMAGE_TAG} ${FRONTEND_ECR_URI}:${IMAGE_TAG}
            echo "Logging into ECR..."
            aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
            echo "Pushing frontend image..."
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
            # build backend binary or docker image depending on repo
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
      agent {
        label 'linux'
      }
      steps {
        withEnv(["PATH+SONAR=${SCANNER_HOME}/bin:${env.PATH}"]) {
          dir('.') {
            sh '''
              echo "Running Sonar Scanner..."
              # Example sonar-scanner invocation - adapt projectKey/name and server details
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

    /*
     * OWASP Dependency-Check Stage
     * --------------------------------------------------
     * You requested this stage to be commented out. If you want to re-enable it later,
     * remove the surrounding comment markers (/* ... *\/) and provide the proper tool image or plugin setup.
     *
     * Example that is currently disabled:
     *
    stage('OWASP Dependency-Check') {
      agent { label 'linux' }
      steps {
        dir('.') {
          sh '''
            echo "Running OWASP Dependency-Check (currently disabled)..."
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
    */

    stage('Notify / Deploy') {
      agent { label 'linux' }
      steps {
        echo "Deployment step placeholder - implement your deployment (ECS, Kubernetes, etc.) here."
        // Example: trigger Terraform, kubectl apply, ecs deploy, etc.
      }
    }
  }

  post {
    success {
      echo "Pipeline succeeded - ${env.BUILD_URL}"
    }
    unstable {
      echo "Pipeline unstable - check warnings"
    }
    failure {
      echo "Pipeline failed - ${env.BUILD_URL}"
    }
    always {
      // Optional: cleanup docker images on agent
      sh '''
        echo "Cleaning up local docker images (if any)..."
        # docker image prune -f || true
      '''
    }
  }
}
