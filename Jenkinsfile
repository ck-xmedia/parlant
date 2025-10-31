pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    APP_PORT   = '8080'
    VENV_DIR   = "${env.WORKSPACE}/.venv"
    LOG_DIR    = "${env.WORKSPACE}/logs"
    PID_FILE   = "${env.WORKSPACE}/run/app.pid"
    NODE_ENV   = 'production'
    PYTHONUNBUFFERED = '1'
    PIP_NO_INPUT = '1'
    PIP_DISABLE_PIP_VERSION_CHECK = '1'
  }

  stages {
    stage('Checkout') {
      steps {
        deleteDir()
        git branch: 'jenkins-automation-28', url: 'https://github.com/ck-xmedia/parlant.git'
      }
    }

    stage('Environment Setup') {
      steps {
        sh '''
          set -eu
          [ -n "${BASH:-}" ] && set -o pipefail
          echo "Setting up environment..."
          mkdir -p "$LOG_DIR" "$(dirname "$PID_FILE")"

          echo "Workspace: $WORKSPACE"
          echo "APP_PORT=$APP_PORT"
          echo "VENV_DIR=$VENV_DIR"

          # Python
          if command -v python3 >/dev/null 2>&1; then
            PY=python3
          elif command -v python >/dev/null 2>&1; then
            PY=python
          else
            PY=""
          fi

          if [ -n "${PY:-}" ]; then
            echo "Python: $($PY -V 2>&1 || true)"
            $PY -m venv "$VENV_DIR" || true
            . "$VENV_DIR/bin/activate" || true
            python -m pip install --upgrade pip setuptools wheel || true
          else
            echo "No Python interpreter found in PATH."
          fi

          # Node.js
          if command -v node >/dev/null 2>&1; then node -v || true; else echo "Node not found."; fi
          if command -v npm >/dev/null 2>&1; then npm -v || true; else echo "npm not found."; fi
          if command -v yarn >/dev/null 2>&1; then yarn -v || true; fi
          if command -v pnpm >/dev/null 2>&1; then pnpm -v || true; fi

          # Java
          if command -v java >/dev/null 2>&1; then java -version || true; else echo "Java not found."; fi
          if command -v mvn  >/dev/null 2>&1; then mvn -v || true; fi
          if command -v gradle >/dev/null 2>&1; then gradle -v || true; fi

          # Other toolchains
          if command -v go >/dev/null 2>&1; then go version || true; fi
          if command -v cargo >/dev/null 2>&1; then cargo --version || true; fi
          if command -v composer >/dev/null 2>&1; then composer --version || true; fi
          if command -v bundle   >/dev/null 2>&1; then bundle --version   || true; fi
        '''
      }
    }

    stage('Install Dependencies') {
      steps {
        sh '''
          set -eu
          [ -n "${BASH:-}" ] && set -o pipefail
          echo "Detecting and installing dependencies..."

          install_python_deps() {
            if [ -f "requirements.txt" ]; then
              echo "[Python] requirements.txt detected"
              . "$VENV_DIR/bin/activate" 2>/dev/null || true
              pip install -r requirements.txt
              return
            fi
            if [ -f "pyproject.toml" ]; then
              echo "[Python] pyproject.toml detected"
              . "$VENV_DIR/bin/activate" 2>/dev/null || true
              if grep -qi "\\[tool.poetry\\]" pyproject.toml && command -v poetry >/dev/null 2>&1; then
                poetry install --no-root --only main --no-interaction --no-ansi
              else
                if [ -f "setup.cfg" ] || [ -f "setup.py" ]; then
                  pip install -e .
                else
                  pip install .
                fi
              fi
              return
            fi
            echo "[Python] No Python dependency files found."
          }

          install_node_deps() {
            if [ -f "package.json" ]; then
              echo "[Node] package.json detected"
              if command -v pnpm >/dev/null 2>&1 && [ -f "pnpm-lock.yaml" ]; then
                pnpm install --frozen-lockfile
              elif command -v yarn >/dev/null 2>&1 && [ -f "yarn.lock" ]; then
                yarn install --frozen-lockfile
              elif command -v npm >/dev/null 2>&1; then
                if [ -f "package-lock.json" ]; then
                  npm ci
                else
                  npm install --no-audit --no-fund
                fi
              else
                echo "[Node] No npm/yarn/pnpm found."
              fi
              return
            fi
            echo "[Node] No package.json found."
          }

          install_java_deps() {
            if [ -f "pom.xml" ]; then
              echo "[Java] Maven project detected"
              mvn -B -U -DskipTests dependency:resolve || true
              return
            fi
            if [ -f "build.gradle" ] || [ -f "build.gradle.kts" ]; then
              echo "[Java] Gradle project detected"
              if [ -x "./gradlew" ]; then
                ./gradlew --no-daemon dependencies || true
              elif command -v gradle >/dev/null 2>&1; then
                gradle --no-daemon dependencies || true
              fi
              return
            fi
            echo "[Java] No Maven/Gradle files found."
          }

          install_go_deps() {
            if [ -f "go.mod" ]; then
              echo "[Go] go.mod detected"
              go mod download
              return
            fi
            echo "[Go] No go.mod found."
          }

          install_rust_deps() {
            if [ -f "Cargo.toml" ]; then
              echo "[Rust] Cargo.toml detected"
              cargo fetch
              return
            fi
            echo "[Rust] No Cargo.toml found."
          }

          install_ruby_deps() {
            if [ -f "Gemfile" ]; then
              echo "[Ruby] Gemfile detected"
              if command -v bundle >/dev/null 2>&1; then
                bundle install --jobs=4 --retry=3
              else
                echo "[Ruby] Bundler not found."
              fi
              return
            fi
            echo "[Ruby] No Gemfile found."
          }

          install_php_deps() {
            if [ -f "composer.json" ]; then
              echo "[PHP] composer.json detected"
              if command -v composer >/dev/null 2>&1; then
                composer install --no-interaction --no-progress --prefer-dist
              else
                echo "[PHP] Composer not found."
              fi
              return
            fi
            echo "[PHP] No composer.json found."
          }

          install_python_deps || { echo "Python dependency step failed" | tee -a "$LOG_DIR/install.log"; }
          install_node_deps   || { echo "Node dependency step failed"   | tee -a "$LOG_DIR/install.log"; }
          install_java_deps   || { echo "Java dependency step failed"   | tee -a "$LOG_DIR/install.log"; }
          install_go_deps     || { echo "Go dependency step failed"     | tee -a "$LOG_DIR/install.log"; }
          install_rust_deps   || { echo "Rust dependency step failed"   | tee -a "$LOG_DIR/install.log"; }
          install_ruby_deps   || { echo "Ruby dependency step failed"   | tee -a "$LOG_DIR/install.log"; }
          install_php_deps    || { echo "PHP dependency step failed"    | tee -a "$LOG_DIR/install.log"; }

          echo "Dependency installation completed."
        '''
      }
    }

    stage('Build') {
      steps {
        sh '''
          set -eu
          [ -n "${BASH:-}" ] && set -o pipefail
          echo "Building project if applicable..."

          build_node() {
            if [ -f "package.json" ]; then
              if grep -q '"build"[[:space:]]*:' package.json; then
                echo "[Node] Running build script"
                if command -v pnpm >/dev/null 2>&1 && [ -f pnpm-lock.yaml ]; then pnpm build
                elif command -v yarn >/dev/null 2>&1 && [ -f yarn.lock ]; then yarn build
                elif command -v npm  >/dev/null 2>&1; then npm run build
                else echo "[Node] No package manager available for build."; fi
              fi
            fi
          }

          build_java() {
            if [ -f "pom.xml" ]; then
              echo "[Java] mvn package"
              mvn -B -DskipTests package
            elif [ -f "build.gradle" ] || [ -f "build.gradle.kts" ]; then
              echo "[Java] gradle build"
              if [ -x "./gradlew" ]; then
                ./gradlew --no-daemon build -x test
              elif command -v gradle >/dev/null 2>&1; then
                gradle --no-daemon build -x test
              fi
            fi
          }

          build_go() {
            if [ -f "go.mod" ]; then
              echo "[Go] Building binary"
              go build -o appbin ./...
            fi
          }

          build_rust() {
            if [ -f "Cargo.toml" ]; then
              echo "[Rust] cargo build --release"
              cargo build --release
            fi
          }

          build_node || true
          build_java || true
          build_go   || true
          build_rust || true
        '''
      }
    }

    stage('Deploy Application') {
      steps {
        sh '''
          set -eu
          [ -n "${BASH:-}" ] && set -o pipefail

          on_error() {
            echo "Deployment failed" | tee -a "$LOG_DIR/deploy.err"
            exit 1
          }
          trap on_error EXIT

          kill_port() {
            port="$1"
            echo "Killing any process on port ${port}..."
            if command -v fuser >/dev/null 2>&1; then
              fuser -k "${port}/tcp" || true
            fi
            if command -v lsof >/dev/null 2>&1; then
              pids="$(lsof -ti :"$port" 2>/dev/null || true)"
              if [ -n "$pids" ]; then
                echo "$pids" | xargs kill -9 || true
              fi
            fi
            if [ -f "$PID_FILE" ]; then
              if ps -p "$(cat "$PID_FILE")" >/dev/null 2>&1; then
                kill -9 "$(cat "$PID_FILE")" || true
              fi
              rm -f "$PID_FILE"
            fi
          }

          start_with_nohup() {
            cmd="$1"
            echo "Starting application: $cmd"
            nohup sh -c "$cmd" >> "$LOG_DIR/app.out" 2>> "$LOG_DIR/app.err" &
            echo $! > "$PID_FILE"
            sleep 2
            if ps -p "$(cat "$PID_FILE")" >/dev/null 2>&1; then
              echo "Application started with PID $(cat "$PID_FILE")"
            else
              echo "Application failed to start" | tee -a "$LOG_DIR/deploy.err"
              exit 1
            fi
          }

          has_npm_script() {
            if [ ! -f package.json ]; then return 1; fi
            if command -v node >/dev/null 2>&1; then
              node -e "try{process.exit(!(require('./package.json').scripts||{}).hasOwnProperty('start'))}catch(e){process.exit(1)}"
              return $?
            fi
            if command -v python3 >/dev/null 2>&1; then
              python3 - << 'PY'
import json,sys
try:
  d=json.load(open('package.json'))
  sys.exit(0 if 'scripts' in d and 'start' in d['scripts'] else 1)
except Exception:
  sys.exit(1)
PY
              return $?
            fi
            grep -q '"start"[[:space:]]*:' package.json
          }

          detect_node_start() {
            if [ -f "package.json" ] && has_npm_script; then
              if command -v pnpm >/dev/null 2>&1 && [ -f pnpm-lock.yaml ]; then
                echo "pnpm start"
                return 0
              elif command -v yarn >/dev/null 2>&1 && [ -f yarn.lock ]; then
                echo "yarn start"
                return 0
              elif command -v npm >/dev/null 2>&1; then
                echo "npm run start"
                return 0
              fi
            fi
            if [ -f "server.js" ]; then echo "node server.js"; return 0; fi
            if [ -f "index.js" ];  then echo "node index.js";  return 0; fi
            return 1
          }

          detect_python_start() {
            . "$VENV_DIR/bin/activate" 2>/dev/null || true
            if [ -f "requirements.txt" ] || [ -f "pyproject.toml" ]; then
              if grep -qiE "uvicorn|fastapi" requirements.txt 2>/dev/null || grep -qiE "uvicorn|fastapi" pyproject.toml 2>/dev/null; then
                if [ -f "app.py" ]; then
                  echo "uvicorn app:app --host 0.0.0.0 --port $APP_PORT"
                  return 0
                fi
              fi
              if grep -qiE "gunicorn|flask" requirements.txt 2>/dev/null || grep -qiE "gunicorn|flask" pyproject.toml 2>/dev/null; then
                if [ -f "wsgi.py" ]; then
                  echo "gunicorn wsgi:app --bind 0.0.0.0:$APP_PORT --workers 2"
                  return 0
                fi
                if [ -f "app.py" ]; then
                  echo "gunicorn app:app --bind 0.0.0.0:$APP_PORT --workers 2"
                  return 0
                fi
              fi
              if [ -f "app.py" ];  then echo "python app.py";  return 0; fi
              if [ -f "main.py" ]; then echo "python main.py"; return 0; fi
            fi
            return 1
          }

          detect_java_start() {
            if [ -f "pom.xml" ]; then
              JAR="$(ls -1 target/*.jar 2>/dev/null | head -n1 || true)"
              if [ -n "$JAR" ]; then
                echo "java -jar \"$JAR\" --server.port=$APP_PORT"
                return 0
              fi
            fi
            if [ -f "build.gradle" ] || [ -f "build.gradle.kts" ]; then
              JAR="$(ls -1 build/libs/*.jar 2>/dev/null | head -n1 || true)"
              if [ -n "$JAR" ]; then
                echo "java -jar \"$JAR\" --server.port=$APP_PORT"
                return 0
              fi
            fi
            return 1
          }

          detect_go_start() {
            if [ -x "./appbin" ]; then echo "./appbin"; return 0; fi
            if [ -f "main.go" ]; then echo "go run ."; return 0; fi
            return 1
          }

          detect_rust_start() {
            if [ -f "Cargo.toml" ]; then
              if [ -x "target/release/$(basename "$(pwd)")" ]; then
                echo "target/release/$(basename "$(pwd)")"
                return 0
              else
                echo "cargo run --release"
                return 0
              fi
            fi
            return 1
          }

          detect_php_start() {
            if [ -f "composer.json" ]; then
              if [ -d "public" ]; then
                echo "php -S 0.0.0.0:$APP_PORT -t public"
              else
                echo "php -S 0.0.0.0:$APP_PORT"
              fi
              return 0
            fi
            return 1
          }

          detect_static_start() {
            for d in dist build public; do
              if [ -d "$d" ]; then
                echo "python -m http.server $APP_PORT --directory \"$d\""
                return 0
              fi
            done
            return 1
          }

          detect_start_cmd() {
            detect_node_start   && return 0
            detect_python_start && return 0
            detect_java_start   && return 0
            detect_go_start     && return 0
            detect_rust_start   && return 0
            detect_php_start    && return 0
            detect_static_start && return 0
            return 1
          }

          kill_port "$APP_PORT"

          CMD="$(detect_start_cmd || true)"
          if [ -z "$CMD" ]; then
            echo "Could not auto-detect a start command for this repository." | tee -a "$LOG_DIR/deploy.err"
            exit 1
          fi

          echo "Selected start command: $CMD" | tee -a "$LOG_DIR/deploy.log"
          start_with_nohup "$CMD"
          trap - EXIT

          echo "Deployment complete. Service should be reachable on port $APP_PORT."
        '''
      }
    }
  }

  post {
    always {
      sh '''
        set -e
        echo "Logs directory contents:"
        ls -la "$LOG_DIR" || true
      '''
      archiveArtifacts artifacts: 'logs/**', allowEmptyArchive: true
    }
    success {
      echo 'Build succeeded, application started in background with nohup.'
    }
    failure {
      echo 'Build failed. Check logs in logs/.'
    }
  }
}