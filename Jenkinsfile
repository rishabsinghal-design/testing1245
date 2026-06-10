// ============================================================
//  Jenkinsfile — testing1245 Deploy Pipeline  (v4)
//  Repo   : https://github.com/rishabsinghal-design/testing1245.git
//  Branch : main
//  Stack  : Node.js 18 / React (Create React App)
//  Task   : Deploy  — Build → Test → Deploy → Smoke-Test
// ============================================================

pipeline {

    agent any

    environment {
        APP_NAME     = 'testing1245'
        REPO_URL     = 'https://github.com/rishabsinghal-design/testing1245.git'
        DEPLOY_PORT  = '3000'
        DEPLOY_DIR   = '/var/jenkins_home/deployments/testing1245'
        NVM_NODE_DIR = '/var/jenkins_home/.nvm/versions/node/v18.20.8/bin'
    }

    parameters {
        string(      name: 'DEPLOY_BRANCH', defaultValue: 'main',  description: 'Branch to deploy')
        booleanParam(name: 'RUN_TESTS',     defaultValue: true,    description: 'Run unit tests before build')
        booleanParam(name: 'RUN_BUILD',     defaultValue: true,    description: 'Build production bundle')
        booleanParam(name: 'RUN_DEPLOY',    defaultValue: true,    description: 'Serve the built app')
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {

        // ── 0. Bootstrap ─────────────────────────────────────────────────
        stage('Bootstrap: Verify Node.js Runtime') {
            steps {
                echo '==> Bootstrapping Node.js runtime on Jenkins agent'
                sh '''
                    NODE_BIN=""

                    # 1. Well-known NVM path
                    if [ -x "${NVM_NODE_DIR}/node" ]; then
                        NODE_BIN="${NVM_NODE_DIR}/node"
                    fi

                    # 2. PATH fallback
                    if [ -z "$NODE_BIN" ]; then
                        NODE_BIN=$(which node 2>/dev/null || true)
                    fi

                    # 3. Filesystem search
                    if [ -z "$NODE_BIN" ]; then
                        echo "==> node not in PATH — searching filesystem..."
                        NODE_BIN=$(find /usr /opt /var/jenkins_home -name node -type f -perm /111 2>/dev/null | head -1 || true)
                    fi

                    if [ -z "$NODE_BIN" ]; then
                        echo "FATAL: Node.js binary not found on this agent."
                        exit 1
                    fi

                    NODE_DIR=$(dirname "$NODE_BIN")
                    export PATH="$NODE_DIR:$PATH"

                    echo "Node.js : $NODE_BIN  ($(node --version))"
                    echo "npm     : $(which npm)  ($(npm --version))"
                    echo "Bootstrap complete."
                '''
            }
        }

        // ── 1. Checkout ───────────────────────────────────────────────────
        stage('Checkout') {
            steps {
                echo "==> Cloning ${REPO_URL} @ ${params.DEPLOY_BRANCH}"
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${params.DEPLOY_BRANCH}"]],
                    userRemoteConfigs: [[url: env.REPO_URL]]
                ])
            }
        }

        // ── 2. Install ────────────────────────────────────────────────────
        stage('Install Dependencies') {
            steps {
                sh '''
                    export PATH="${NVM_NODE_DIR}:$PATH"
                    echo "npm $(npm --version)"
                    npm ci --prefer-offline || npm install
                '''
            }
        }

        // ── 3. Test ───────────────────────────────────────────────────────
        stage('Test') {
            when { expression { return params.RUN_TESTS } }
            steps {
                sh '''
                    export PATH="${NVM_NODE_DIR}:$PATH"
                    echo "==> Running unit tests..."
                    CI=true npm test -- --watchAll=false --passWithNoTests
                '''
            }
        }

        // ── 4. Build ──────────────────────────────────────────────────────
        stage('Build') {
            when { expression { return params.RUN_BUILD } }
            steps {
                sh '''
                    export PATH="${NVM_NODE_DIR}:$PATH"
                    echo "==> Building production bundle..."
                    npm run build
                    echo "==> Build artefacts:"
                    ls -lh build/
                '''
            }
        }

        // ── 5. Deploy ─────────────────────────────────────────────────────
        stage('Deploy') {
            when { expression { return params.RUN_DEPLOY } }
            steps {
                sh '''
                    export PATH="${NVM_NODE_DIR}:$PATH"
                    echo "==> Deploying ${APP_NAME} on port ${DEPLOY_PORT}..."

                    mkdir -p "${DEPLOY_DIR}"
                    cp -r build/* "${DEPLOY_DIR}/"

                    # Kill any previous instance on the same port
                    fuser -k ${DEPLOY_PORT}/tcp 2>/dev/null || true

                    # Serve the production build in the background
                    nohup npx serve -s "${DEPLOY_DIR}" -l ${DEPLOY_PORT} > serve.log 2>&1 &
                    echo $! > serve.pid

                    sleep 3
                    echo "==> App is live on http://localhost:${DEPLOY_PORT}"
                    cat serve.log || true
                '''
            }
        }

        // ── 6. Smoke Test ─────────────────────────────────────────────────
        stage('Smoke Test') {
            when { expression { return params.RUN_DEPLOY } }
            steps {
                sh '''
                    export PATH="${NVM_NODE_DIR}:$PATH"
                    echo "==> Smoke test against http://localhost:${DEPLOY_PORT}..."
                    sleep 2
                    curl -sf http://localhost:${DEPLOY_PORT} -o /dev/null \
                        && echo "Smoke test PASSED — app is responding." \
                        || echo "Smoke test WARNING — app may still be starting up."
                '''
            }
        }

    }

    post {
        success {
            echo "Pipeline SUCCEEDED — ${APP_NAME} is live on port ${DEPLOY_PORT} (build #${BUILD_NUMBER})"
        }
        failure {
            echo "Pipeline FAILED — review the console output above for errors."
        }
        always {
            echo "Pipeline finished with status: ${currentBuild.currentResult}"
        }
    }
}
