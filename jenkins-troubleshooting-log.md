# Jenkins Pipeline & Two-Tier Flask App — Complete Setup, Errors, Fixes & Jenkinsfile

A single reference file containing all starting-to-ending commands for the Flask app, every Jenkins pipeline error encountered across runs, how each issue was resolved, and the final working `Jenkinsfile`.

---

## 1. Starting-to-Ending Commands (Flask App & Server)

### A. Server & Jenkins Permission Setup (Start)
```bash
### Check Jenkins service status and user group membership
sudo systemctl status jenkins
id jenkins

# Grant Jenkins user access to Docker socket (/var/run/docker.sock) and restart Jenkins
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
sudo systemctl is-active jenkins

# Optional: Install python3.14-venv (if creating local Python virtualenvs on Ubuntu/Debian)
sudo apt-get update && sudo apt-get install -y python3.14-venv

# Optional: Install docker-compose-plugin if using `docker compose` on the host
sudo apt-get update && sudo apt-get install -y docker-compose-plugin
docker compose version

### Clone, Build Image, Test & Run Two-Tier App (Core Flow)

# 1. Clone the repository
git clone -b master [https://github.com/LondheShubham153/two-tier-flask-app.git](https://github.com/LondheShubham153/two-tier-flask-app.git)
cd two-tier-flask-app

# 2. Build the Docker image locally
docker build -t two-tier-flask-app .
docker images | grep two-tier-flask-app

# 3. Run tests or smoke-check the app module
pytest || python3 -m pytest
docker run --rm --entrypoint python3 two-tier-flask-app -c "import app; print('app module import OK')"

# 4. Create the Docker network and run MySQL + Flask App containers (app runs on port 5000)
docker network create twotier || true
docker rm -f flask-app mysql || true

docker run -d --name mysql --network twotier \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=devops \
  -e MYSQL_USER=admin \
  -e MYSQL_PASSWORD=admin \
  mysql:8.4

docker run -d --name flask-app --network twotier -p 5000:5000 \
  -e MYSQL_HOST=mysql \
  -e MYSQL_USER=admin \
  -e MYSQL_PASSWORD=admin \
  -e MYSQL_DB=devops \
  two-tier-flask-app

  #Verification, Logs & Cleanup Commands

### Check running containers and port mapping (0.0.0.0:5000->5000/tcp)
docker ps
docker ps -a

# Inspect container logs if a container exits or fails
docker logs mysql
docker logs flask-app

# Verify health endpoint locally and via server IP
curl -i http://localhost:5000/health
curl -i [http://13.201.87.182:5000/health](http://13.201.87.182:5000/health)

# Stop and remove containers and network when done
docker rm -f flask-app mysql || true
docker network rm twotier || true

# All Issues Encountered & How They Were Solved

## Two-Tier Flask App (`two-tier`)

### 1. Missing Docker Hub Credentials (`ERROR: dockerhub-creds`)
* **Issue:** Pipeline failed before running stages because `dockerhub-creds` did not exist.
* **Fix:** Removed `dockerhub-creds` and Docker Hub push steps; built the image locally from the cloned Git repository instead.

### 2. `sh` Inside `post` Without `node` Context (`MissingContextVariableException: hudson.FilePath`)
* **Issue:** Running `sh` in `post` without a node context failed with a missing `FilePath` context.
* **Fix:** Wrapped the `post` shell step inside a `node(...)` block.

### 3. Undefined `BRANCH_NAME` / `REPO_URL` (`MissingPropertyException`)
* **Issue:** `git` step referenced `${BRANCH_NAME}` and `${REPO_URL}` without defining them.
* **Fix:** Defined `REPO_URL` and `BRANCH` in `environment` (or passed `'master'` and `'https://github.com/LondheShubham153/two-tier-flask-app.git'` directly).

### 4. Docker Socket Permission Denied (`unix:///var/run/docker.sock`)
* **Issue:** `jenkins` user lacked permission to access the Docker daemon socket.
* **Fix:** Ran `sudo usermod -aG docker jenkins` and `sudo systemctl restart jenkins`.

### 5. Missing `python3.14-venv` (`ensurepip` Not Available)
* **Issue:** `python3 -m venv .venv` failed because `python3.14-venv` was missing.
* **Fix:** Installed `python3.14-venv` on the host or ran `pytest` / container module import checks in stage 3 without `.venv`.

### 6. Test Stage Exit Code 5 (`NO TESTS RAN` / `pytest` Not in Image)
* **Issue:** `pytest` wasn't installed in the image and `unittest discover` found `0` tests (exit code `5`).
* **Fix:** Checked for `pytest` / `python3 -m pytest` and fell back to `docker run --rm --entrypoint python3 ... -c "import app; print('app module import OK')"`.

### 7. Compilation Error on `node { ... }` (`Missing required parameter: "label"`)
* **Issue:** Calling `node { ... }` without `label` caused a Declarative Pipeline compilation error.
* **Fix:** Used `node(label: '') { ... }`.

### 8. Docker Compose Flag / Missing Compose Binary (`unknown shorthand flag: 'd' in -d` / `Neither docker compose nor docker-compose found`)
* **Issue:** Compose subcommand/flag was unavailable or incompatible on the host.
* **Fix:** Created the `twotier` network and ran `mysql` (`mysql:8.4`) and `flask-app` (port `5000`) directly via `docker run` (or installed `docker-compose-plugin`).

---

## Spring Boot Backend (`easy-curd`)

### 9. `mvn: not found`
* **Issue:** Maven was not installed or not on `PATH` on the build node (`cd backend && mvn clean package`).
* **Fix:** Installed Maven on the node running the build, linked `mvn` into `/usr/local/bin` if needed, and restarted Jenkins / reconnected the agent.

### 10. Java Release Version 17 Not Supported (`error: release version 17 not supported`)
* **Issue:** Active `javac` was older than Java 17.
* **Fix:** Installed `openjdk-17-jdk`, selected Java 17 via `update-alternatives`, and restarted Jenkins.

### 11. Spring Boot Context Failure (`Failed to load ApplicationContext` / `Unable to determine Dialect`)
* **Issue:** Tests tried to connect to a database not present during packaging.
* **Fix:** Built with `mvn clean package -DskipTests` (or provided datasource/test DB configuration), producing the `.jar` artifact cleanly.

