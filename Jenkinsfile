pipeline {
  agent any

  environment {
    NODE_VERSION = '18'
    APP_NAME     = 'testing1245'
    REPO_URL     = 'https://github.com/rishabsinghal-design/testing1245.git'
    DEPLOY_PORT  = '3000'
  }

  options {
    buildDiscarder(logRotator(numToKeepStr: '10'))
    timestamps()
    timeout(time: 20, unit: 'MINUTES')
  }

  stages {

    stage('Checkout') {
      steps {
        echo "Cloning ${REPO_URL} @ main"
        git branch: 'main', url: "${REPO_URL}"
      }
    }

    stage('Install Dependencies') {
      steps {
        echo 'Installing npm dependencies...'
        sh 'npm ci --prefer-offline || npm install'
      }
    }

    stage('Test') {
      steps {
        echo 'Running unit tests...'
        sh 'CI=true npm test -- --watchAll=false --passWithNoTests'
      }
    }

    stage('Build') {
      steps {
        echo 'Building production bundle...'
        sh 'npm run build'
        echo 'Build artefacts:'
        sh 'ls -lh build/'
      }
    }

    stage('Deploy') {
      steps {
        echo "Deploying ${APP_NAME} on port ${DEPLOY_PORT}..."
        sh '''
          # Kill any previous instance on the same port
          fuser -k ${DEPLOY_PORT}/tcp || true

          # Serve the production build in the background
          nohup npx serve -s build -l ${DEPLOY_PORT} > serve.log 2>&1 &
          echo $! > serve.pid

          sleep 3
          echo "App is live on http://localhost:${DEPLOY_PORT}"
          cat serve.log || true
        '''
      }
    }

  }

  post {
    success {
      echo "✅ Deployment successful — ${APP_NAME} is running on port ${DEPLOY_PORT}"
    }
    failure {
      echo "❌ Pipeline failed. Check the console output above for details."
    }
    always {
      echo "Pipeline finished with status: ${currentBuild.currentResult}"
    }
  }
}
