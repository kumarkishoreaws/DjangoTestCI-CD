pipeline {
  agent {
    docker {
      image '<your-dockerhub-username>/ci-agent:latest'
      args '-v /var/run/docker.sock:/var/run/docker.sock'
    }
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '30'))
    ansiColor('xterm')
  }

  environment {
    APP_NAME        = 'conduit'           // CHANGE if needed
    EC2_USER        = 'ubuntu'            // CHANGE: EC2 username
    EC2_HOST        = '3.89.119.30' // CHANGE: EC2 public DNS
    DEPLOY_DIR      = '/home/ubuntu/app'  // CHANGE: path on EC2 where code will be deployed
    // GIT_COMMIT_SHORT will be set at runtime
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        script {
          env.GIT_COMMIT_SHORT = sh(returnStdout: true, script: "git rev-parse --short HEAD").trim()
          echo "Commit short hash: ${env.GIT_COMMIT_SHORT}"
        }
      }
    }

    stage('Prepare (dependencies + ensure jq)') {
      steps {
        sh '''
          set -e
          echo "Python version:"
          python3 --version 
          pip3 --version 

          if [ -f requirements.txt ]; then
            echo "Installing Python requirements..."
            pip3 install --upgrade pip setuptools wheel
            pip3 install --no-cache-dir -r requirements.txt
          else
            echo "No requirements.txt found - skipping pip install"
          fi

          if ! command -v jq >/dev/null 2>&1; then
            echo "jq not found - installing jq"
            apt-get update -y
            apt-get install -y --no-install-recommends jq
          else
            echo "jq already present"
          fi
        '''
      }
    }

    stage('Static Analysis') {
      steps {
        sh '''
          set -e
          mkdir -p reports
          echo "Running flake8..."
          flake8 . --exit-zero --output-file=reports/flake8.txt || true
          echo "Running pylint..."
          PY_FILES=$(git ls-files '*.py' || true)
          if [ -n "$PY_FILES" ]; then
            pylint $PY_FILES > reports/pylint.txt || true
          else
            echo "No python files found for pylint" > reports/pylint.txt
          fi
          echo "Running bandit..."
          bandit -r . -f json -o reports/bandit.json || true
        '''
        recordIssues tools: [flake8(pattern: 'reports/flake8.txt'), pylint(pattern: 'reports/pylint.txt'), bandit(pattern: 'reports/bandit.json')]
      }
      post {
        always {
          archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
        }
      }
    }

    stage('Unit Tests & Coverage') {
      steps {
        sh '''
          set -e
          mkdir -p reports htmlcov
          pytest --junitxml=reports/junit.xml --cov=. --cov-report=html:htmlcov || true
        '''
      }
      post {
        always {
          junit keepLongStdio: true, testResults: 'reports/junit.xml'
          publishHTML (target: [
            reportDir: 'htmlcov',
            reportFiles: 'index.html',
            reportName: "Coverage Report",
            allowMissing: true,
            alwaysLinkToLastBuild: true
          ])
          archiveArtifacts artifacts: 'reports/**,htmlcov/**', allowEmptyArchive: true
        }
      }
    }

    stage('Deploy to EC2') {
      steps {
        sshagent (credentials: ['123']) {   // Jenkins credential with your EC2 private key
          sh '''
           set -e
        echo "Deploying to EC2: ${EC2_HOST}"
        
        # Copy source code to EC2
        rsync -avz --delete ./ ${EC2_USER}@${EC2_HOST}:${DEPLOY_DIR}

        # Run remote commands (install deps, migrate)
        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} << 'EOF'
          cd ${DEPLOY_DIR}
          
		  echo "creating env "
          source venv/bin/activate		  
          echo "Installing requirements on EC2..."
          if [ -f requirements.txt ]; then
            pip3 install --no-cache-dir -r requirements.txt
          fi
          
          echo "Applying Django migrations..."
          python3 manage.py migrate --noinput || true
          
          echo "Starting Django development server..."
          # Optional: run in background for testing (will serve on port 8000)
          nohup python3 manage.py runserver 0.0.0.0:8000 &> django.log &
        EOF
        }
      }
    }

  } // stages

  post {
    success {
      echo "Pipeline finished successfully. Code deployed to EC2: ${EC2_HOST}:${DEPLOY_DIR}"
    }
    failure {
      echo "Pipeline failed — check console output and artifact reports."
    }
    always {
      archiveArtifacts artifacts: 'reports/**,htmlcov/**', allowEmptyArchive: true
    }
  }
}
