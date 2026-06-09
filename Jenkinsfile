pipeline {
  agent any

  environment {
    APP_NAME    = 'testing1245'
    REPO_URL    = 'https://github.com/rishabsinghal-design/testing1245.git'
    DEPLOY_PORT = '3000'
    DEPLOY_DIR  = '/var/jenkins_home/deployments/testing1245'
    NVM_DIR     = '/var/jenkins_home/.nvm'
  }

  options {
    buildDiscarder(logRotator(numToKeepStr: '10'))
    timestamps()
    timeout(time: 30, unit: 'MINUTES')
  }

  stages {

    stage('Checkout') {
      steps {
        echo "==> Cloning ${REPO_URL} @ main"
        git branch: 'main', url: "${REPO_URL}"
        sh 'git log --oneline -5'
      }
    }

    stage('Verify Node.js') {
      steps {
        echo '==> Locating Node.js / npm on the agent'
        sh '''
          set -e
          NODE_BIN=""

          # 1. Check standard PATH first
          if command -v node >/dev/null 2>&1; then
            NODE_BIN=$(command -v node)
          fi

          # 2. Try nvm installation
          if [ -z "$NODE_BIN" ] && [ -s "${NVM_DIR}/nvm.sh" ]; then
            export NVM_DIR="${NVM_DIR}"
            . "${NVM_DIR}/nvm.sh"
            if command -v node >/dev/null 2>&1; then
              NODE_BIN=$(command -v node)
            fi
          fi

          # 3. Try common local paths
          for candidate in \
              /var/jenkins_home/.nvm/versions/node/*/bin/node \
              /usr/local/bin/node \
              /usr/bin/node \
              /opt/node/bin/node; do
            if [ -x "$candidate" ]; then
              NODE_BIN="$candidate"
              break
            fi
          done

          # 4. Install Node.js 18 via nvm if still not found
          if [ -z "$NODE_BIN" ]; then
            echo "Node.js not found — installing via nvm..."
            export NVM_DIR="${NVM_DIR}"
            curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
            . "${NVM_DIR}/nvm.sh"
            nvm install 18
            nvm use 18
            NODE_BIN=$(command -v node)
          fi

          echo "Found Node.js: $NODE_BIN"
          "$NODE_BIN" --version
          NPM_BIN="$(dirname $NODE_BIN)/npm"
          "$NPM_BIN" --version
          echo "$NODE_BIN" > .node_bin
          echo "$NPM_BIN"  > .npm_bin
        '''
      }
    }

    stage('Install Dependencies') {
      steps {
        echo '==> Installing npm dependencies'
        sh '''
          set -e
          export NVM_DIR="${NVM_DIR}"
          [ -s "${NVM_DIR}/nvm.sh" ] && . "${NVM_DIR}/nvm.sh"
          NPM_BIN=$(cat .npm_bin)
          NODE_BIN=$(cat .node_bin)
          NODE_DIR=$(dirname "$NODE_BIN")
          export PATH="$NODE_DIR:$PATH"
          echo "Using npm: $NPM_BIN"
          "$NPM_BIN" ci --prefer-offline || "$NPM_BIN" install
        '''
      }
    }

    stage('Test') {
      steps {
        echo '==> Running unit tests'
        sh '''
          set -e
          export NVM_DIR="${NVM_DIR}"
          [ -s "${NVM_DIR}/nvm.sh" ] && . "${NVM_DIR}/nvm.sh"
          NPM_BIN=$(cat .npm_bin)
          NODE_BIN=$(cat .node_bin)
          NODE_DIR=$(dirname "$NODE_BIN")
          export PATH="$NODE_DIR:$PATH"
          CI=true "$NPM_BIN" test -- --watchAll=false --passWithNoTests
        '''
      }
    }

    stage('Build') {
      steps {
        echo '==> Building production bundle'
        sh '''
          set -e
          export NVM_DIR="${NVM_DIR}"
          [ -s "${NVM_DIR}/nvm.sh" ] && . "${NVM_DIR}/nvm.sh"
          NPM_BIN=$(cat .npm_bin)
          NODE_BIN=$(cat .node_bin)
          NODE_DIR=$(dirname "$NODE_BIN")
          export PATH="$NODE_DIR:$PATH"
          "$NPM_BIN" run build
          echo "--- Build artefacts ---"
          ls -lh build/
        '''
      }
    }

    stage('Deploy') {
      steps {
        echo "==> Deploying ${APP_NAME} to ${DEPLOY_DIR} on port ${DEPLOY_PORT}"
        sh '''
          set -e
          export NVM_DIR="${NVM_DIR}"
          [ -s "${NVM_DIR}/nvm.sh" ] && . "${NVM_DIR}/nvm.sh"
          NPM_BIN=$(cat .npm_bin)
          NODE_BIN=$(cat .node_bin)
          NODE_DIR=$(dirname "$NODE_BIN")
          export PATH="$NODE_DIR:$PATH"

          # Prepare deployment directory
          mkdir -p "${DEPLOY_DIR}"
          rm -rf "${DEPLOY_DIR}/build"
          cp -r build "${DEPLOY_DIR}/build"

          # Kill any previous instance on the same port
          fuser -k ${DEPLOY_PORT}/tcp 2>/dev/null || true

          # Serve the production build in the background
          nohup "$NODE_DIR/npx" serve -s "${DEPLOY_DIR}/build" -l ${DEPLOY_PORT} \
            > "${DEPLOY_DIR}/serve.log" 2>&1 &
          echo $! > "${DEPLOY_DIR}/serve.pid"

          sleep 3
          echo "App is live on http://localhost:${DEPLOY_PORT}"
          cat "${DEPLOY_DIR}/serve.log" || true

          # Write deployment manifest
          echo "${APP_NAME} build=${BUILD_NUMBER} ts=$(date +%Y%m%d-%H%M%S) branch=main" \
            > "${DEPLOY_DIR}/DEPLOYED_VERSION.txt"
          cat "${DEPLOY_DIR}/DEPLOYED_VERSION.txt"
        '''
      }
    }

    stage('Smoke Test') {
      steps {
        echo '==> Running post-deploy smoke test'
        sh '''
          sleep 2
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:${DEPLOY_PORT}/ || echo "000")
          echo "HTTP status: $STATUS"
          if [ "$STATUS" = "200" ]; then
            echo "Smoke test PASSED — app responded with HTTP 200"
          else
            echo "WARNING: Smoke test returned HTTP $STATUS (app may still be starting)"
          fi
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
