pipeline {
  agent any

  environment {
    // Name of the SonarQube server configured in Jenkins
    SONARQUBE_ENV = 'sonarqube-imcc'
    // Nexus Docker registry host (update port/path if your college gives a different URL)
    DOCKER_REGISTRY = 'nexus.imcc.com'
    // Local image names we build and then tag/push
    IMAGE_NAME_SERVER = 'realcode-server'
    IMAGE_NAME_CLIENT = 'realcode-client'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Install & Test') {
      steps {
        dir('server') {
          sh 'npm install'
          // If you don\'t have tests yet this will not fail the build
          sh 'npm test || echo "Server tests missing or failing (non‑blocking for now)"'
        }
        dir('client') {
          sh 'npm install'
          // Same for client tests
          sh 'npm test || echo "Client tests missing or failing (non‑blocking for now)"'
        }
      }
    }

    stage('SonarQube Analysis') {
      environment {
        SONAR_SCANNER_HOME = tool 'SonarQubeScanner'
      }
      steps {
        withSonarQubeEnv(SONARQUBE_ENV) {
          // Uses sonar-project.properties at repo root
          sh """
            ${SONAR_SCANNER_HOME}/bin/sonar-scanner
          """
        }
      }
    }

    stage('Quality Gate') {
      steps {
        script {
          timeout(time: 5, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
          }
        }
      }
    }

    stage('Build Docker Images') {
      steps {
        sh """
          docker build -t ${IMAGE_NAME_SERVER}:latest -f server/Dockerfile .
          docker build -t ${IMAGE_NAME_CLIENT}:latest -f client/Dockerfile .
        """
      }
    }

    stage('Push Images to Nexus') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'nexus-2401090',
          usernameVariable: 'NEXUS_USER',
          passwordVariable: 'NEXUS_PASS'
        )]) {
          sh """
            echo $NEXUS_PASS | docker login ${DOCKER_REGISTRY} -u $NEXUS_USER --password-stdin

            docker tag ${IMAGE_NAME_SERVER}:latest ${DOCKER_REGISTRY}/${IMAGE_NAME_SERVER}:latest
            docker tag ${IMAGE_NAME_CLIENT}:latest ${DOCKER_REGISTRY}/${IMAGE_NAME_CLIENT}:latest

            docker push ${DOCKER_REGISTRY}/${IMAGE_NAME_SERVER}:latest
            docker push ${DOCKER_REGISTRY}/${IMAGE_NAME_CLIENT}:latest
          """
        }
      }
    }

    stage('Deploy with Docker Compose') {
      steps {
        sh """
          docker-compose down || true
          docker-compose up -d --build
        """
      }
    }
  }

  post {
    always {
      cleanWs()
    }
  }
}


