pipeline {
    agent {
        docker {
            image 'kichu2320/ephemeral-image:V1'
            args '-v /var/run/docker.sock:/var/run/docker.sock -u 0:0'
        }
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '30'))
        ansiColor('xterm')
    }

    environment {
        APP_NAME   = 'conduit'
        EC2_USER   = 'ubuntu'
        EC2_HOST   = '54.226.77.122'
        DEPLOY_DIR = '/home/ubuntu/app'
    }

    stages {

        stage('Checkout') {
            steps {
                sh "git config --global --add safe.directory $WORKSPACE"
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        returnStdout: true,
                        script: "git rev-parse --short HEAD"
                    ).trim()
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
            mkdir -p reports

            echo "Running flake8..."
            flake8 . --exit-zero --output-file=reports/flake8.txt || true

            echo "Running pylint..."
            PY_FILES=$(git ls-files '*.py' || true)
            if [ -n "$PY_FILES" ]; then
                pylint $PY_FILES --output-format=text > reports/pylint.txt || true
            else
                echo "No python files found for pylint" > reports/pylint.txt
            fi

            echo "Running bandit..."
            bandit -r . -f json -o reports/bandit.json || true
        '''
        
        // Collect issues with Warnings Next Generation plugin
       // recordIssues tools: [
         //   flake8(pattern: 'reports/flake8.txt'),
           // pylint(pattern: 'reports/pylint.txt'),
            // bandit(pattern: 'reports/bandit.json')
        // ]
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
                   
                    mkdir -p reports htmlcov
                    pytest --junitxml=reports/junit.xml --cov=. --cov-report=html:htmlcov || true
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, keepLongStdio: true, testResults: 'reports/junit.xml'
                    publishHTML(target: [
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
    agent {
        docker {
            image 'kichu2320/cd-image:V1'
            reuseNode true
        }
    }
    steps {
         sshagent(credentials: ['123']) {
            sh '''
                echo "Deploying to EC2: ${EC2_HOST}"
 
                # Ensure .ssh folder exists
                mkdir -p ~/.ssh
                chmod 700 ~/.ssh

                # Add EC2 host to known_hosts
                ssh-keyscan -H ${EC2_HOST} > ~/.ssh/known_hosts
                chmod 644 ~/.ssh/known_hosts

 
                rsync -avz --delete ./ ${EC2_USER}@${EC2_HOST}:${DEPLOY_DIR}

                ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} << "ENDSSH"
                    cd ${DEPLOY_DIR}
                    echo "Activating virtual environment..."
                    [ ! -d venv ] && python3 -m venv venv
                    source venv/bin/activate
                    if [ -f requirements.txt ]; then
                        pip3 install --no-cache-dir -r requirements.txt
                    fi
                    echo "Applying Django migrations..."
                    python3 manage.py migrate --noinput || true
                    echo "Starting Django server..."
                    pkill -f "manage.py runserver" || true
                    nohup python3 manage.py runserver 0.0.0.0:8000 &> django.log &
                ENDSSH
                 
            '''
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
