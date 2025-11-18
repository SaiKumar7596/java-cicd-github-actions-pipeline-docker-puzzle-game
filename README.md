
# Puzzle Game CI/CD with GitHub Actions, SonarQube, Nexus, Docker & Tomcat

This repository demonstrates a **complete CI/CD pipeline** for a Java web application using:

- **GitHub Actions** for CI/CD
- **SonarQube** for code quality analysis
- **Nexus** for Maven artifact storage
- **Docker & Docker Hub** for container images
- **Tomcat running on EC2** for deployment

On every push to `main`, the pipeline:

1. Clones the source code
2. Runs **SonarQube** analysis
3. Builds the app using **Maven**
4. Uploads the WAR to **Nexus**
5. Builds a **custom Tomcat Docker image**
6. Pushes the image to **Docker Hub**
7. SSHs into **EC2** and deploys the updated container

---

## 1. What This Project Is

A fully automated **CI/CD system for a Java web app** where:

- **Builds & tests** run in **GitHub Actions**
- **Quality gates** are enforced by **SonarQube**
- **Artifacts** are stored in **Nexus**
- **Runtime** is a **Dockerized Tomcat app** on an **AWS EC2** instance

### Why It Matters

- Shows **real-world CI/CD** using industry tools
- Covers the **full lifecycle**: code → quality → artifact → image → deployment
- Great for **interviews**, **DevOps practice**, and **portfolio projects**

---

## 2. High-Level Architecture

```text
Developer Pushes Code (GitHub)
             |
             v
+----------------------------+
|      GitHub Actions        |
|  .github/workflows/ci-cd  |
+-------------+--------------+
              |
   +----------+-----------+----------------------------+
   |                      |                            |
   v                      v                            v
SonarQube           Maven Build & Deploy          Docker Build & Push
(EC2:9000)         to Nexus (EC2:8081)           to Docker Hub
                                                  |
                                                  v
                                         +------------------+
                                         |  EC2 Deployment  |
                                         |  (Tomcat Docker) |
                                         +------------------+
                                                  |
                                                  v
                                    http://<EC2-IP>:8080/
````

---

## 3. Tech Stack

* **Runtime:** AWS EC2 (Ubuntu)
* **CI/CD:** GitHub Actions
* **Language:** Java (with Maven)
* **App Server:** Tomcat (Docker container)
* **Code Quality:** SonarQube (Docker container on EC2)
* **Artifact Repo:** Nexus (Docker container on EC2)
* **Container Registry:** Docker Hub

---

## 4. Prerequisites

* AWS account + EC2 instance (Ubuntu)
* GitHub repository (this source code)
* Docker Hub account
* Basic understanding of:

  * Docker
  * Maven
  * SSH / Linux

---

## 5. Create EC2 Instance (Ubuntu)

1. **Launch EC2 Instance**

   * OS: **Ubuntu**
   * Instance type: **t2.large**
   * Storage: **16 GB**
   * Security group:

     * Allow **SSH (22)**
     * Allow **HTTP (80)** (optional)
     * Allow **8080** (Tomcat)
     * Allow **8081** (Nexus)
     * Allow **9000** (SonarQube)
     * (For lab/demo you may allow all inbound, but not recommended for prod)

2. **Create / Download key pair** (e.g. `your-key.pem`).

---

## 6. SSH into EC2 & Initial Setup

```bash
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```

Update server:

```bash
sudo apt update -y
sudo apt upgrade -y
```

Clone the source code:

```bash
git clone https://github.com/SaiKumar7596/puzzle-game-java
cd puzzle-game-java
```

---

## 7. Install Docker on EC2

```bash
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ubuntu
```

Log out and log in again (or run `newgrp docker`) to apply group changes.

---

## 8. Run SonarQube in Docker

Pull SonarQube image:

```bash
docker pull sonarqube
```

Run SonarQube container:

```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube
```

Access in browser:

```text
http://<EC2_PUBLIC_IP>:9000
```

Default login:

* Username: `admin`
* Password: `admin`

Change the password on first login, then:

* Create a **SonarQube token**
* Save it (will be used as `SONAR_TOKEN` in GitHub Secrets)

---

## 9. Install Java 17 & Maven on EC2 (for local checks)

```bash
sudo apt install openjdk-17-jdk -y
sudo apt install maven -y
```

Verify:

```bash
java -version
mvn -version
```

(Optional) Run a local Sonar scan to confirm:

```bash
mvn clean install sonar:sonar \
  -Dsonar.host.url=http://<EC2_PUBLIC_IP>:9000 \
  -Dsonar.login=<YOUR_SONAR_TOKEN>
```

---

## 10. Run Nexus Repository in Docker

Pull Nexus:

```bash
docker pull sonatype/nexus3
```

Run Nexus:

```bash
docker run -d --name nexus -p 8081:8081 sonatype/nexus3
```

Access in browser:

```text
http://<EC2_PUBLIC_IP>:8081/
```

Get default admin password:

```bash
docker exec -it nexus cat /nexus-data/admin.password
```

Login → Change password → Create / confirm `maven-releases` repository.

---

## 11. Configure Maven for Nexus

### Update `pom.xml`

Inside your project `pom.xml`, add:

```xml
<distributionManagement>
  <repository>
    <id>maven-releases</id>
    <name>Maven Releases</name>
    <url>http://<EC2_PUBLIC_IP>:8081/repository/maven-releases/</url>
  </repository>
</distributionManagement>
```

### Create Maven `settings.xml` (local)

Create or edit:

```bash
vi ~/.m2/settings.xml
```

Add:

```xml
<settings>
  <servers>
    <server>
      <id>maven-releases</id>
      <username>admin</username>
      <password>your_nexus_password</password>
    </server>
  </servers>
</settings>
```

---

## 12. Test Deploy to Nexus (Local)

```bash
mvn clean deploy
```

Check Nexus:

```text
http://<EC2_PUBLIC_IP>:8081/
```

Browse to:

```text
maven-releases/com/example/puzzle-game-webapp/1.0/
```

You should see: `puzzle-game-webapp-1.0.war`.

---

## 13. Create Custom Tomcat Docker Image

Create `Dockerfile` in the project root:

```dockerfile
# Step 1: Use official Tomcat image with JDK 17
FROM tomcat:10.1.10-jdk17

# Step 2: Switch to root and install curl
USER root
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*

# Step 3: Remove default Tomcat webapps
RUN rm -rf /usr/local/tomcat/webapps/*

# Step 4: Set environment variables for Nexus
ENV NEXUS_USER=admin
ENV NEXUS_PASS=admin123

# WAR location in Nexus
ENV WAR_URL=http://<EC2_PUBLIC_IP>:8081/repository/maven-releases/com/example/puzzle-game-webapp/1.0/puzzle-game-webapp-1.0.war
ENV WAR_NAME=puzzle-game-webapp.war

# Step 5: Set working directory
WORKDIR /usr/local/tomcat/webapps

# Step 6: Download WAR
RUN echo "Downloading WAR from Nexus..." && \
    curl -u "$NEXUS_USER:$NEXUS_PASS" -L -f -o "$WAR_NAME" "$WAR_URL" && \
    echo "WAR downloaded successfully:" && ls -lh $WAR_NAME

# Step 7: Expose Tomcat port
EXPOSE 8080

# Step 8: Start Tomcat
CMD ["catalina.sh", "run"]
```

Build:

```bash
docker build -t pz-tomcat:1.0 .
```

Test run:

```bash
docker run -d --name tomcat -p 8080:8080 pz-tomcat:1.0
```

Visit:

```text
http://<EC2_PUBLIC_IP>:8080/puzzle-game-webapp/
```

---

## 14. Push Image to Docker Hub

Log in:

```bash
docker login
```

Tag & push:

```bash
docker tag pz-tomcat:1.0 saikumar7596/my-repo:1.0
docker push saikumar7596/my-repo:1.0
```

(We will use this repo name in GitHub Actions.)

---

## 15. GitHub Actions CI/CD Overview

We now move the **entire pipeline** into **GitHub Actions**:

* Trigger: push to `main`
* Runner: `ubuntu-latest` (GitHub-hosted)
* Steps:

  1. Checkout code
  2. Set up JDK + Maven
  3. Run SonarQube scan
  4. Deploy WAR to Nexus
  5. Build Docker image
  6. Push to Docker Hub
  7. SSH into EC2 and deploy new container

---

## 16. Required GitHub Secrets

Go to:

> GitHub → Repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

Create these secrets:

| Secret Name       | Description                 | Example                    |
| ----------------- | --------------------------- | -------------------------- |
| `SONAR_HOST_URL`  | SonarQube URL               | `http://54.81.139.84:9000` |
| `SONAR_TOKEN`     | SonarQube token             | `squ_...`                  |
| `NEXUS_USER`      | Nexus username              | `admin`                    |
| `NEXUS_PASS`      | Nexus password              | `admin123`                 |
| `DOCKERHUB_USER`  | Docker Hub username         | `saikumar7596`             |
| `DOCKERHUB_TOKEN` | Docker Hub PAT/token        | `*****`                    |
| `SSH_PRIVATE_KEY` | EC2 SSH private key content | contents of `your-key.pem` |
| `TOMCAT_USER`     | (Optional) Tomcat user      | `admin`                    |
| `TOMCAT_PASS`     | (Optional) Tomcat password  | `admin123`                 |

---

## 17. GitHub Actions CI/CD Workflow

Create this file in your repo:

**`.github/workflows/ci-cd.yml`**

```yaml
name: CI-CD Pipeline

on:
  push:
    branches: [ "main" ]

env:
  EC2_HOST: 54.81.139.84
  DOCKER_IMAGE_NAME: saikumar7596/my-repo

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Cache Maven packages
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      - name: Configure Maven settings for Nexus
        run: |
          mkdir -p ~/.m2
          cat > ~/.m2/settings.xml <<EOF
          <settings>
            <servers>
              <server>
                <id>maven-releases</id>
                <username>${NEXUS_USER}</username>
                <password>${NEXUS_PASS}</password>
              </server>
            </servers>
          </settings>
          EOF
        env:
          NEXUS_USER: ${{ secrets.NEXUS_USER }}
          NEXUS_PASS: ${{ secrets.NEXUS_PASS }}

      - name: SonarQube Analysis + Maven Build
        run: |
          mvn -B clean verify sonar:sonar \
            -Dsonar.host.url=${{ secrets.SONAR_HOST_URL }} \
            -Dsonar.login=${{ secrets.SONAR_TOKEN }}
      
      - name: Deploy artifact to Nexus
        run: mvn -B deploy

      - name: Log in to Docker Hub
        run: echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login -u "${{ secrets.DOCKERHUB_USER }}" --password-stdin

      - name: Build Docker image
        run: |
          IMAGE_TAG=${GITHUB_SHA::7}
          docker build -t $DOCKER_IMAGE_NAME:$IMAGE_TAG .
          docker tag $DOCKER_IMAGE_NAME:$IMAGE_TAG $DOCKER_IMAGE_NAME:latest
        env:
          DOCKER_IMAGE_NAME: ${{ env.DOCKER_IMAGE_NAME }}

      - name: Push Docker image
        run: |
          IMAGE_TAG=${GITHUB_SHA::7}
          docker push $DOCKER_IMAGE_NAME:$IMAGE_TAG
          docker push $DOCKER_IMAGE_NAME:latest
        env:
          DOCKER_IMAGE_NAME: ${{ env.DOCKER_IMAGE_NAME }}

      - name: Prepare SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${SSH_PRIVATE_KEY}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan -H $EC2_HOST >> ~/.ssh/known_hosts
        env:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
          EC2_HOST: ${{ env.EC2_HOST }}

      - name: Deploy to EC2 (Tomcat container)
        run: |
          IMAGE_TAG=${GITHUB_SHA::7}
          ssh ubuntu@$EC2_HOST "
            docker rm -f tomcat || true &&
            docker pull $DOCKER_IMAGE_NAME:latest &&
            docker run -d --name tomcat -p 8080:8080 $DOCKER_IMAGE_NAME:latest
          "
        env:
          EC2_HOST: ${{ env.EC2_HOST }}
          DOCKER_IMAGE_NAME: ${{ env.DOCKER_IMAGE_NAME }}
```

### Workflow Summary

* **Checkout source** from GitHub.
* Configure Maven to talk to **Nexus** using GitHub Secrets.
* Run **SonarQube analysis** + Maven **build**.
* Deploy WAR to **Nexus**.
* Build **Docker image**, tag with:

  * `latest`
  * short commit SHA (e.g. `dee5d43`)
* Push image to **Docker Hub**.
* SSH into **EC2** using `SSH_PRIVATE_KEY` and:

  * Stop old Tomcat container (if any)
  * Pull new Docker image
  * Run new Tomcat container on port `8080`.

---

## 18. Verification Steps

### 1️⃣ GitHub Actions Workflow Success

* Go to: **GitHub → Your Repo → Actions**
* Select the latest run of `CI-CD Pipeline`
* You should see:

  * Green **✓ Success**
  * All steps passed (SonarQube, Build, Nexus Upload, Docker Push, Deploy)

**(Later) Screenshot placeholder:**

```markdown
![GitHub Actions - CI/CD Success](images/github-actions-success.png)
*CI/CD workflow succeeded with all jobs green.*
```

---

### 2️⃣ Nexus Repository – WAR Uploaded

Go to:

```text
http://54.81.139.84:8081/
```

Navigate:

```text
Browse → maven-releases → com/example/puzzle-game-webapp/1.0/
```

You should see:

```text
puzzle-game-webapp-1.0.war
```

**Screenshot placeholder:**

```markdown
![Nexus - WAR Artifact](images/nexus-war-upload.png)
*WAR file successfully stored under maven-releases/com/example/puzzle-game-webapp/1.0/.*
```

---

### 3️⃣ Docker Hub – Image Pushed

Go to:

```text
https://hub.docker.com/repository/docker/saikumar7596/my-repo
```

You should see tags like:

* `latest`
* `<short-commit-sha>` (e.g. `dee5d43`)

**Screenshot placeholder:**

```markdown
![Docker Hub - Image Tags](images/dockerhub-image-tags.png)
*Docker image pushed with both latest and commit-specific tags.*
```

---

### 4️⃣ EC2 – Running Updated Container

On EC2:

```bash
ssh -i your-key.pem ubuntu@54.81.139.84
docker ps
```

You should see:

```text
CONTAINER ID   IMAGE                             NAMES
...            saikumar7596/my-repo:latest       tomcat
```

And in browser:

```text
http://54.81.139.84:8080/
```

(or `http://54.81.139.84:8080/puzzle-game-webapp/` depending on WAR context path)

**Screenshot placeholder:**

```markdown
![EC2 - Tomcat Container Running](images/ec2-docker-ps.png)
*Tomcat container running with the latest Docker image deployed from GitHub Actions.*
```

---

## 19. Troubleshooting

### 🔴 SonarQube Step Fails

* Check `SONAR_HOST_URL` & `SONAR_TOKEN` in GitHub Secrets.
* Confirm SonarQube is running:

  ```bash
  docker ps | grep sonarqube
  ```
* Check if EC2 security group allows port `9000`.

### 🔴 Nexus Deploy Fails

* Check Maven `distributionManagement` in `pom.xml`.
* Confirm `NEXUS_USER` / `NEXUS_PASS` in GitHub Secrets.
* Verify Nexus is reachable from the public internet:

  ```text
  http://54.81.139.84:8081/
  ```

### 🔴 Docker Push Fails

* Confirm `DOCKERHUB_USER` & `DOCKERHUB_TOKEN` are correct.
* Ensure token has **write** permission to Docker Hub registry.

### 🔴 SSH Deploy Step Fails

* Ensure `SSH_PRIVATE_KEY` matches the EC2 key pair.
* Confirm EC2 security group allows SSH (22) from GitHub runners (or 0.0.0.0/0 in lab).
* You can test locally:

  ```bash
  ssh -i your-key.pem ubuntu@54.81.139.84
  ```

### 🔴 App Not Accessible on Browser

* Check container:

  ```bash
  docker ps
  docker logs tomcat
  ```
* Confirm port `8080` is open in EC2 Security Group.
* Try URLs:

  ```text
  http://54.81.139.84:8080/
  http://54.81.139.84:8080/puzzle-game-webapp/
  ```

---

## 20. What This Pipeline Demonstrates (Interview-Ready Summary)

> **“I built a complete CI/CD pipeline for a Java web application using GitHub Actions, SonarQube, Nexus, Docker, and Tomcat on AWS EC2.
> On every push to main, GitHub Actions checks out the code, runs a SonarQube analysis, builds the artifact with Maven, deploys it to a Nexus repository, builds and pushes a Docker image to Docker Hub, and finally connects to an EC2 instance over SSH to deploy the latest container. The entire workflow is driven by GitHub Secrets for SonarQube, Nexus, Docker Hub, and SSH credentials.”**

You can use that as a concise **interview answer** describing this project.

---

## 21. Next Steps / Possible Enhancements

* Add **branch protections** & run CI for PRs.
* Add **staging vs production** environments.
* Use **GitHub Environments** with approvals for prod deploy.
* Add automated **tests** and enforce **Quality Gate** from SonarQube in the pipeline.

---

```

If you want, next I can:

- Customize the workflow to **multi-job** (build → scan → deploy separated),
- Or add a **rollback strategy** using previous Docker tags / Nexus artifacts.
```
