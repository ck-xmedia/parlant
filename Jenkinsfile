pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '20'))
    timeout(time: 30, unit: 'MINUTES')
  }

  environment {
    REPO_URL     = 'https://github.com/ck-xmedia/parlant.git'
    BRANCH_NAME  = 'jenkins-automation-27'
    APP_PORT     = '8080'
    VENV_DIR     = "${WORKSPACE}/.venv"
    LOG_DIR      = "${WORKSPACE}/logs"
    RUN_DIR      = "${WORKSPACE}/run"
    APP_PID_FILE = "${WORKSPACE}/run/app.pid"
    NODE_ENV     = 'production'
  }

  stages {
    stage('Checkout') {
      steps {
        deleteDir()
        git branch: "${BRANCH_NAME}", url: "${REPO_URL}"
      }
    }

    stage('Environment Setup') {
      steps {
        sh '''
          set -euxo pipefail
          mkdir -p "$LOG_DIR" "$RUN_DIR"
          : > "$LOG_DIR/pipeline.log"

          echo "Initializing environment" | tee -a "$LOG_DIR/pipeline.log"
          echo "WORKSPACE: $WORKSPACE" | tee -a "$LOG_DIR/pipeline.log"
          echo "APP_PORT: $APP_PORT" | tee -a "$LOG_DIR/pipeline.log"

          # Show available runtimes
          (command -v python3 && python3 --version) || true
          (command -v python && python --version) || true
          (command -v pip && pip --version) || true
          (command -v node && node --version) || true
          (command -v npm && npm --version) || true
          (command -v mvn && mvn -v) || true
          (command -v gradle && gradle -v) || true
          (command -v go && go version) || true
          (command -v cargo && cargo --version) || true

          echo "Environment setup complete" | tee -a "$LOG_DIR/pipeline.log"
        '''
      }
    }

    stage('Install Dependencies') {
      steps {
        sh '''
          set -euo pipefail
          echo "Detecting project type and installing dependencies..." | tee -a "$LOG_DIR/pipeline.log"

          PROJECT_TYPE="UNKNOWN"

          detect_python_deps() {
            if [ -f "pyproject.toml" ] || [ -f "requirements.txt" ] || ls requirements*.txt 1>/dev/null 2>&1 || [ -f "Pipfile" ] || [ -f "poetry.lock" ]; then
              return 0
            fi
            return 1
          }

          detect_node_deps() {
            [ -f "package.json" ]
          }

          detect_maven_deps() {
            [ -f "pom.xml" ]
          }

          detect_gradle_deps() {
            [ -f "build.gradle" ] || [ -f "settings.gradle" ] || [ -f "gradlew" ]
          }

          detect_go_deps() {
            [ -f "go.mod" ]
          }

          detect_rust_deps() {
            [ -f "Cargo.toml" ]
          }

          if detect_python_deps; then
            PROJECT_TYPE="PYTHON"
            echo "$PROJECT_TYPE" > "$RUN_DIR/project.type"
            echo "Detected Python project" | tee -a "$LOG_DIR/pipeline.log"

            PYTHON_BIN="$(command -v python3 || true)"
            [ -n "$PYTHON_BIN" ] || PYTHON_BIN="$(command -v python || true)"
            if [ -z "$PYTHON_BIN" ]; then
              echo "Python 3 is required but not found on the agent." | tee -a "$LOG_DIR/pipeline.log"
              exit 1
            fi

            "$PYTHON_BIN" -m venv "$VENV_DIR"
            . "$VENV_DIR/bin/activate"
            python -m pip install --upgrade pip wheel setuptools | tee -a "$LOG_DIR/pipeline.log"

            if ls requirements*.txt 1>/dev/null 2>&1; then
              REQ_FILE="$(ls -1 requirements*.txt | head -n1)"
              echo "Installing dependencies from $REQ_FILE" | tee -a "$LOG_DIR/pipeline.log"
              pip install -r "$REQ_FILE" | tee -a "$LOG_DIR/pipeline.log"
            elif [ -f "pyproject.toml" ]; then
              echo "Installing project from pyproject.toml" | tee -a "$LOG_DIR/pipeline.log"
              pip install . | tee -a "$LOG_DIR/pipeline.log"
            elif [ -f "Pipfile" ]; then
              echo "Installing via Pipenv" | tee -a "$LOG_DIR/pipeline.log"
              pip install pipenv | tee -a "$LOG_DIR/pipeline.log"
              pipenv install --system --deploy | tee -a "$LOG_DIR/pipeline.log"
            else
              echo "No explicit dependencies file; installing project" | tee -a "$LOG_DIR/pipeline.log"
              pip install . | tee -a "$LOG_DIR/pipeline.log"
            fi

            pip freeze > "$LOG_DIR/requirements.freeze.txt" || true

          elif detect_node_deps; then
            PROJECT_TYPE="NODE"
            echo "$PROJECT_TYPE" > "$RUN_DIR/project.type"
            echo "Detected Node.js project" | tee -a "$LOG_DIR/pipeline.log"
            if [ -f "package-lock.json" ]; then
              npm ci --prefer-offline --no-audit --no-fund | tee -a "$LOG_DIR/pipeline.log"
            else
              npm install --no-audit --no-fund | tee -a "$LOG_DIR/pipeline.log"
            fi

            # Build if script exists
            if grep -q '"build"' package.json; then
              npm run build | tee -a "$LOG_DIR/pipeline.log"
            fi

          elif detect_maven_deps; then
            PROJECT_TYPE="MAVEN"
            echo "$PROJECT_TYPE" > "$RUN_DIR/project.type"
            echo "Detected Maven project" | tee -a "$LOG_DIR/pipeline.log"
            mvn -B -ntp -Dmaven.test.skip=true clean package | tee -a "$LOG_DIR/pipeline.log"

          elif detect_gradle_deps; then
            PROJECT_TYPE="GRADLE"
            echo "$PROJECT_TYPE" > "$RUN_DIR/project.type"
            echo "Detected Gradle project" | tee -a "$LOG_DIR/pipeline.log"
            if [ -x "./gradlew" ]; then
              ./gradlew clean build -x test | tee -a "$LOG_DIR/pipeline.log"
            else
              gradle clean build -x test | tee -a "$LOG_DIR/pipeline.log"
            fi

          elif detect_go_deps; then
            PROJECT_TYPE="GO"
            echo "$PROJECT_TYPE" > "$RUN_DIR/project.type"
            echo "Detected Go project" | tee -a "$LOG_DIR/pipeline.log"
            go mod download | tee -a "$LOG_DIR/pipeline.log"
            mkdir -p bin
            go build -o bin/app ./... | tee -a "$LOG_DIR/pipeline.log"

          elif detect_rust_deps; then
            PROJECT_TYPE="RUST"
            echo "$PROJECT_TYPE" > "$RUN_DIR/project.type"
            echo "Detected Rust project" | tee -a "$LOG_DIR/pipeline.log"
            cargo build --release | tee -a "$LOG_DIR/pipeline.log"

          else
            PROJECT_TYPE="UNKNOWN"
            echo "$PROJECT_TYPE" > "$RUN_DIR/project.type"
            echo "No known dependency files detected. Proceeding with generic deployment flow." | tee -a "$LOG_DIR/pipeline.log"
          fi

          echo "Project type: $PROJECT_TYPE" | tee -a "$LOG_DIR/pipeline.log"
        '''
      }
    }

    stage('Deploy Application') {
      steps {
        sh '''
          set -euo pipefail
          echo "Starting deployment on port ${APP_PORT}" | tee -a "$LOG_DIR/pipeline.log"
          mkdir -p "$RUN_DIR"

          # Kill any previous PID we launched
          if [ -f "$APP_PID_FILE" ]; then
            OLD_PID="$(cat "$APP_PID_FILE" || true)"
            if [ -n "${OLD_PID:-}" ] && ps -p "$OLD_PID" >/dev/null 2>&1; then
              echo "Killing previous app process by PID: $OLD_PID" | tee -a "$LOG_DIR/pipeline.log"
              kill -9 "$OLD_PID" || true
            fi
            rm -f "$APP_PID_FILE"
          fi

          # Kill anything on the desired port
          if command -v lsof >/dev/null 2>&1; then
            PIDS="$(lsof -t -i tcp:${APP_PORT} || true)"
            if [ -n "${PIDS:-}" ]; then
              echo "Killing processes on port ${APP_PORT}: $PIDS" | tee -a "$LOG_DIR/pipeline.log"
              kill -9 $PIDS || true
            fi
          fi

          if command -v fuser >/dev/null 2>&1; then
            fuser -k -n tcp ${APP_PORT} || true
          fi

          TYPE="$(cat "$RUN_DIR/project.type")"
          echo "Deploying project type: $TYPE" | tee -a "$LOG_DIR/pipeline.log"

          start_python() {
            PYTHON_BIN="$(command -v python3 || true)"
            [ -n "$PYTHON_BIN" ] || PYTHON_BIN="$(command -v python || true)"
            if [ -z "$PYTHON_BIN" ]; then
              echo "Python not found; cannot deploy Python app" | tee -a "$LOG_DIR/pipeline.log"
              exit 1
            fi

            if [ -d "$VENV_DIR" ]; then
              . "$VENV_DIR/bin/activate"
            fi

            if python -c "import uvicorn" 2>/dev/null; then
              APP_MODULE=""
              if [ -f "app.py" ] && grep -qiE "FastAPI|Starlette" app.py; then APP_MODULE="app:app"; fi
              if [ -z "$APP_MODULE" ] && [ -f "main.py" ] && grep -qiE "FastAPI|Starlette" main.py; then APP_MODULE="main:app"; fi
              if [ -z "$APP_MODULE" ]; then APP_MODULE="app:app"; fi
              echo "Starting Uvicorn ${APP_MODULE} on ${APP_PORT}" | tee -a "$LOG_DIR/pipeline.log"
              nohup uvicorn "$APP_MODULE" --host 0.0.0.0 --port ${APP_PORT} > "$LOG_DIR/app.out" 2>&1 &
              echo $! > "$APP_PID_FILE"
            elif python -c "import flask" 2>/dev/null; then
              export FLASK_APP="${FLASK_APP:-app.py}"
              export FLASK_RUN_PORT="${APP_PORT}"
              export FLASK_RUN_HOST="0.0.0.0"
              echo "Starting Flask app ${FLASK_APP}" | tee -a "$LOG_DIR/pipeline.log"
              nohup flask run > "$LOG_DIR/app.out" 2>&1 &
              echo $! > "$APP_PID_FILE"
            else
              echo "No known Python web framework detected; serving current directory" | tee -a "$LOG_DIR/pipeline.log"
              nohup "$PYTHON_BIN" -m http.server ${APP_PORT} > "$LOG_DIR/app.out" 2>&1 &
              echo $! > "$APP_PID_FILE"
            fi
          }

          start_node() {
            if [ -f "package.json" ] && grep -q '"start"' package.json; then
              echo "Starting Node with npm start" | tee -a "$LOG_DIR/pipeline.log"
              nohup npm run start > "$LOG_DIR/app.out" 2>&1 &
              echo $! > "$APP_PID_FILE"
            elif [ -f "server.js" ]; then
              echo "Starting Node server.js" | tee -a "$LOG_DIR/pipeline.log"
              nohup node server.js > "$LOG_DIR/app.out" 2>&1 &
              echo $! > "$APP_PID_FILE"
            else
              echo "No Node start script; attempting static server via npx http-server or Python fallback" | tee -a "$LOG_DIR/pipeline.log"
              if command -v npx >/dev/null 2>&1; then
                nohup npx http-server -p ${APP_PORT} > "$LOG_DIR/app.out" 2>&1 &
                echo $! > "$APP_PID_FILE"
              else
                PY="$(command -v python3 || command -v python || true)"
                nohup "$PY" -m http.server ${APP_PORT} > "$LOG_DIR/app.out" 2>&1 &
                echo $! > "$APP_PID_FILE"
              fi
            fi
          }

          start_maven() {
            ART="$(ls -1 target/*.jar 2>/dev/null | head -n1 || true)"
            if [ -n "$ART" ]; then
              echo "Starting JAR $ART on ${APP_PORT}" | tee -a "$LOG_DIR/pipeline.log"
              nohup java -jar "$ART" --server.port=${APP_PORT} > "$LOG_DIR/app.out" 2>&1 &
              echo $! > "$APP_PID_FILE"
            else
              echo "No built JAR found in target/, cannot deploy" | tee -a "$LOG_DIR/pipeline.log"
              exit 1
            fi
          }

          start_gradle() {
            ART="$(ls -1 build/libs/*.jar 2>/dev/null | head -n1 || true)"
            if [ -n "$ART" ]; then
              echo "Starting JAR $ART on ${APP_PORT}" | tee -a "$LOG_DIR/pipeline.log"
              nohup java -jar "$ART" --server.port=${APP_PORT} > "$LOG_DIR/app.out" 2>&1 &
              echo $! > "$APP_PID_FILE"
            else
              echo "No built JAR found in build/libs/, cannot deploy" | tee -a "$LOG_DIR/pipeline.log"
              exit 1
            fi
          }

          start_go() {
            if [ -x "bin/app" ]; then
              echo "Starting Go binary on ${APP_PORT}" | tee -a "$LOG_DIR/pipeline.log"
              nohup ./bin/app -port ${APP_PORT} > "$LOG_DIR/app.out" 2>&1 &
              echo $! > "$APP_PID_FILE"
            else
              echo "Go binary bin/app not found; cannot deploy" | tee -a "$LOG_DIR/pipeline.log"
              exit 1
            fi
          }

          start_rust() {
            BIN="$(ls -1 target/release/* 2>/dev/null | head -n1 || true)"
            if [ -n "$BIN" ] && [ -x "$BIN" ]; then
              echo "Starting Rust binary on ${APP_PORT}" | tee -a "$LOG_DIR/pipeline.log"
              nohup "$BIN" --port ${APP_PORT} > "$LOG_DIR/app.out" 2>&1 &
              echo $! > "$APP_PID_FILE"
            else
              echo "Rust binary not found in target/release; cannot deploy" | tee -a "$LOG_DIR/pipeline.log"
              exit 1
            fi
          }

          case "$TYPE" in
            PYTHON) start_python ;;
            NODE)   start_node ;;
            MAVEN)  start_maven ;;
            GRADLE) start_gradle ;;
            GO)     start_go ;;
            RUST)   start_rust ;;
            *)      echo "Unknown project type; serving current directory on ${APP_PORT}" | tee -a "$LOG_DIR/pipeline.log"
                    PY="$(command -v python3 || command -v python || true)"
                    nohup "$PY" -m http.server ${APP_PORT} > "$LOG_DIR/app.out" 2>&1 &
                    echo $! > "$APP_PID_FILE"
                    ;;
          esac

          sleep 2

          # Verify port is listening
          if command -v ss >/dev/null 2>&1; then
            ss -ltnp | grep ":${APP_PORT}" || { echo "Application is not listening on port ${APP_PORT}" | tee -a "$LOG_DIR/pipeline.log"; exit 1; }
          elif command -v netstat >/dev/null 2>&1; then
            netstat -ltnp | grep ":${APP_PORT}" || { echo "Application is not listening on port ${APP_PORT}" | tee -a "$LOG_DIR/pipeline.log"; exit 1; }
          else
            echo "No ss/netstat available to verify listening port; proceeding." | tee -a "$LOG_DIR/pipeline.log"
          fi

          echo "Deployment completed. PID: $(cat "$APP_PID_FILE")" | tee -a "$LOG_DIR/pipeline.log"
        '''
      }
    }
  }

  post {
    success {
      echo "Build and deployment successful."
      archiveArtifacts artifacts: 'logs/**', fingerprint: true, allowEmptyArchive: true
    }
    failure {
      echo "Build or deployment failed."
      archiveArtifacts artifacts: 'logs/**', fingerprint: true, allowEmptyArchive: true
    }
    always {
      script {
        currentBuild.description = "Branch: ${env.BRANCH_NAME}, Port: ${env.APP_PORT}"
        // Intentionally do not kill the app; it remains running on ${APP_PORT}.
      }
    }
  }
}