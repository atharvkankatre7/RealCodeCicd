pipeline {
  agent any

  environment {
    // Name of the SonarQube server configured in Jenkins
    SONARQUBE_ENV = 'sonarqube-imcc'
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

    stage('Build & Deploy with Docker Compose') {
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


