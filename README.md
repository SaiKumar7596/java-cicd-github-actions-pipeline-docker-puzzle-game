
# 🌐 **CI/CD Pipeline with GitHub Actions, SonarQube, Nexus, Docker & EC2**

*End-to-End Production-Style DevOps Project*

---

## 📌 **1. Project Overview**

This project implements a **complete CI/CD pipeline** for a Java Web Application using:

| Tool                         | Purpose                                   |
| ---------------------------- | ----------------------------------------- |
| **EC2 (t2.large)**           | Host for SonarQube, Nexus, Docker, Tomcat |
| **Docker**                   | Run SonarQube, Nexus & Tomcat             |
| **GitHub Actions**           | Full CI/CD automation                     |
| **SonarQube**                | Code quality analysis                     |
| **Nexus Repository Manager** | Store WAR artifacts                       |
| **DockerHub**                | Store Docker images                       |
| **Tomcat (container)**       | Hosts the deployed application            |

---

# 🖥 **2. EC2 Instance Setup**

### **2.1 Launch EC2 Instance**

* AMI: **Ubuntu 22.04 LTS**
* Instance type: **t2.large** (required for SonarQube)
* Storage: **16 GB** (minimum for Docker + Nexus + Sonar)
* Security group:

  * `22/tcp` (SSH)
  * `8080/tcp` (Tomcat)
  * `8081/tcp` (Nexus)
  * `9000/tcp` (SonarQube)
  * `80/tcp` (web)
  * `443/tcp` (optional)

---

## 📦 **3. Install Docker on EC2**

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ubuntu
```

(Reboot required after this)

```bash
sudo reboot
```

---

# 🐳 **4. Run Required Containers**

---

## **4.1 Start SonarQube Container**

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts
```

Verify:

```bash
curl http://<EC2-IP>:9000/api/system/status
```

Expected:

```
{"status":"UP"}
```

---

## **4.2 Start Nexus Repository Container**

```bash
docker run -d -p 8081:8081 --name nexus sonatype/nexus3
```

Access Nexus:

```
http://<EC2-IP>:8081
```

Create:

* **maven-releases** repo
* Note username/password → store in GitHub secrets

---

## **4.3 Tomcat will be deployed automatically by GitHub Actions**

No need to start Tomcat manually.

---

# 💾 **5. GitHub Secrets Required**

Go to:

```
GitHub → Repo → Settings → Secrets → Actions
```

Add the following:

| Secret Name       | Value                       |
| ----------------- | --------------------------- |
| `SONAR_HOST_URL`  | http://<EC2-IP>:9000        |
| `SONAR_TOKEN`     | Sonar → My Account → Tokens |
| `NEXUS_USER`      | admin                       |
| `NEXUS_PASS`      | your Nexus password         |
| `DOCKERHUB_USER`  | your DockerHub username     |
| `DOCKERHUB_TOKEN` | DockerHub → Access Token    |
| `SSH_PRIVATE_KEY` | key used to SSH into EC2    |

---

# 📁 **6. Project Structure**

```
.
├── src/
├── pom.xml
├── Dockerfile
└── .github/workflows/cicd.yml
```

---

# 🐳 **7. Dockerfile (Final)**

```dockerfile
# Final Tomcat Image
FROM tomcat:10.1.10-jdk17

# Cleanup default apps
RUN rm -rf /usr/local/tomcat/webapps/*

# Copy WAR built by GitHub Actions
COPY target/*.war /usr/local/tomcat/webapps/ROOT.war

EXPOSE 8080
CMD ["catalina.sh", "run"]
```

---

# 🤖 **8. GitHub Actions CI/CD Pipeline (Final Version)**

Create file:

```
.github/workflows/cicd.yml
```

Paste:

```yaml
name: CI-CD Webapp

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    env:
      NEXUS_URL: "http://<EC2-IP>:8081/repository/maven-releases/"
      GROUP_ID_PATH: "com/example"
      APP_NAME: "puzzle-game-webapp"
      DOCKER_REPO: "yourdockerhub/repo"
      DEPLOY_HOST: "<EC2-IP>"
      DEPLOY_USER: "ubuntu"
      CONTAINER_NAME: "tomcat"

    steps:

      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17

      - name: SonarQube Scan
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
        run: |
          mvn -B clean verify sonar:sonar \
            -DskipTests=true \
            -Dsonar.projectKey=${APP_NAME} \
            -Dsonar.host.url=${SONAR_HOST_URL} \
            -Dsonar.login=${SONAR_TOKEN}

      - name: Install jq
        run: sudo apt-get install -y jq

      - name: Check SonarQube Quality Gate
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
        run: |
          STATUS=$(curl -s -u "${SONAR_TOKEN}:" \
            "${SONAR_HOST_URL%/}/api/qualitygates/project_status?projectKey=${APP_NAME}" \
            | jq -r '.projectStatus.status')
          echo "Quality Gate: $STATUS"
          test "$STATUS" = "OK"

      - name: Build WAR
        run: mvn -B clean package

      - name: Upload WAR to Nexus
        env:
          NEXUS_USER: ${{ secrets.NEXUS_USER }}
          NEXUS_PASS: ${{ secrets.NEXUS_PASS }}
        run: |
          WAR=$(ls target/*.war | head -n 1)
          VERSION=$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout)
          ARTIFACT_URL="${NEXUS_URL%/}/${GROUP_ID_PATH}/${APP_NAME}/${VERSION}/${APP_NAME}-${VERSION}.war"
          curl -v -u "${NEXUS_USER}:${NEXUS_PASS}" --upload-file "$WAR" "$ARTIFACT_URL"

      - name: Docker Login
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USER }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and Push Docker Image
        run: |
          SHORT_SHA=${GITHUB_SHA::7}
          docker build -t ${DOCKER_REPO}:${SHORT_SHA} -t ${DOCKER_REPO}:latest .
          docker push ${DOCKER_REPO}:${SHORT_SHA}
          docker push ${DOCKER_REPO}:latest

      - name: Deploy to EC2 Server
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ env.DEPLOY_HOST }}
          username: ${{ env.DEPLOY_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          envs: "DOCKER_REPO, CONTAINER_NAME"
          script: |
            docker pull ${DOCKER_REPO}:latest
            docker stop ${CONTAINER_NAME} 2>/dev/null || true
            docker rm ${CONTAINER_NAME} 2>/dev/null || true
            docker run -d --name ${CONTAINER_NAME} -p 8080:8080 ${DOCKER_REPO}:latest
```

---

# 🚀 **9. Deployment Verification**

### ✔ Check containers on EC2

```bash
docker ps
```

Expected:

```
tomcat  (your app)
sonar
nexus
```

### ✔ Open App in Browser

```
http://<EC2-IP>:8080
```

You should now see the deployed Java web application.

---

# 📸 **10. Screenshot Suggestions (for documentation)**

You should add these screenshots:

### **EC2 Setup**

✔ Instance details page
✔ Security group rules

### **Docker Setup**

✔ docker ps showing sonar + nexus
✔ Sonar UI
✔ Nexus UI

### **GitHub Actions Workflow**

✔ Pipeline success
✔ SonarQube analysis passed
✔ Nexus upload success
✔ DockerHub image uploaded
✔ Deployment step success

### **Application Output**

✔ Webapp running on EC2 port 8080

---

# 🏁 **11. Conclusion**

This lab demonstrates complete CI/CD automation using:

* GitHub Actions
* SonarQube Quality Gate
* Nexus Artifact Storage
* Docker Image Build & Push
* Remote EC2 Deployment
* Automatic Tomcat Deployment

This is a **full production-grade DevOps pipeline** and perfect for:

* GitHub portfolio
* DevOps interviews
* Resume project showcase

---

# 🎯 Want me to create a downloadable **PDF** version of this documentation?

I can generate it with perfect formatting — just say:

👉 **"Yes, generate the PDF"**
