# Currency Converter CI/CD Pipeline (Jenkins + Maven + SonarQube + Nexus + Docker + Tomcat)

This project demonstrates a complete end-to-end **CI/CD pipeline** for a Java-based **Currency Converter Web Application** deployed via **Tomcat (Docker)** using:

- **GitHub** → Source Code Management  
- **Jenkins Pipeline** → CI/CD automation  
- **SonarQube** → Code Quality  
- **Nexus Repository Manager** → Artifact Storage  
- **Docker** → Containerization  
- **DockerHub** → Image Registry  
- **Tomcat 10** → Deployment  
- **AWS EC2 (Ubuntu Linux)** → Server Host  

This README includes **every step**, all configurations, commands, tokens, repo setup, deployment procedure, and pipeline logic used in the project.

---

# 📁 1. Project Structure

```
currency-converter/
│── src/
│── pom.xml
│── Jenkinsfile
│── Dockerfile
└── README.md
```

---

# 🔧 2. Prerequisites to Install (Server Setup)

Run on your Ubuntu EC2 machine:

```bash
sudo hostnamectl set-hostname docker-js
sudo init 6
sudo apt -y update
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo chmod 666 /var/run/docker.sock
```
<img width="1920" height="1080" alt="7" src="https://github.com/user-attachments/assets/0f96c68f-5b7c-4a25-af00-230ec6cf2d46" />

---

# 🔐 3. Personal Access Tokens (PAT)

## 👉 GitHub PAT (required for git push)
Generate:

1. GitHub → Settings  
2. Developer Settings  
3. Personal Access Tokens → Fine-grained token  
4. Permissions:
   - Contents → Read/Write
   - Metadata → Read

Use this during:

```
git push origin main
Username: jagadapi240
Password: ghp_----
```

---

## 👉 DockerHub PAT (required for Jenkins push)
Generate:

1. DockerHub → Settings → Security  
2. Create Access Token  

Add in Jenkins:

- ID: `dockerhub-pat`
- Username: `jagadapi240`
- Password: dckr_pat--------
---
<img width="1920" height="1080" alt="6" src="https://github.com/user-attachments/assets/68233d3e-0b0e-4537-8479-6d0bc2316b1d" />

# 🏢 4. Nexus Setup (Artifact Repository)

### Run Nexus:
```bash
docker run -d --name nexus -p 8081:8081 sonatype/nexus3
```

Open Nexus:
```
http://51.21.202.150:8081/
```

Default password:
```bash
docker exec -it nexus cat /nexus-data/admin.password
```

### Create Repository: `maven-releases`
- Format: Maven2
- Type: Hosted
- **Version Policy: Release**
- **Deployment Policy: Allow redeploy** (important!)

---
<img width="1920" height="1080" alt="3" src="https://github.com/user-attachments/assets/4498262f-8d7c-43df-9867-c297c1898d24" />
# 🔍 5. SonarQube Setup

### Run SonarQube:
```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts
```

Open:
```
http://51.21.202.150:9000/
```

Login:
```
username: admin
password: admin
```
<img width="1920" height="1080" alt="4" src="https://github.com/user-attachments/assets/6be18ba3-e372-4c31-8246-541f853aaf18" />

Generate token:
- My Account → Security → Generate Token

Add in Jenkins as:
- ID: squ_fc9e0d62999e2472efdfaa16a08d06b8ec61c865 `sonar-token`

---

# 🤖 6. Jenkins Setup

### Run Jenkins:
```bash
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```

Get initial admin password:
```bash
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

### Install Jenkins Plugins
- Git
- Pipeline
- Maven Integration
- Docker Pipeline
- SonarQube Scanner
- Credentials Binding
- GitHub Integration

### Configure Tools
Jenkins → Manage Jenkins → Global Tool Configuration:

#### Maven:
```
Name: Maven
Version: Latest
```

#### SonarQube:
```
Name: MySonarQube
URL: http://51.21.202.150:9000/
Token: sonar-token
```

#### Credentials Added:
- `nexus-creds` → Nexus username/password
- `dockerhub-pat` → DockerHub username/token
- `sonar-token` → SonarQube token

---
<img width="1920" height="1080" alt="5" src="https://github.com/user-attachments/assets/2769775a-380e-4e81-bf6e-2edde1a822ae" />

# 🌐 7. GitHub Webhook Setup

GitHub repo → Settings → Webhooks → Add Webhook:

```
Payload URL: http://51.21.202.150:8080/github-webhook/
Content type: application/json
Trigger: Push events
```

---

# 🧩 8. pom.xml (Important Sections)

```xml
<project>
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.ajacs</groupId>
    <artifactId>currency-converter-web</artifactId>
    <version>1.0.1</version>
    <packaging>war</packaging>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>jakarta.servlet</groupId>
            <artifactId>jakarta.servlet-api</artifactId>
            <version>6.0.0</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>

    <distributionManagement>
        <repository>
            <id>nexus</id>
            <url>http://51.21.202.150:8081/repository/maven-releases/</url>
        </repository>
    </distributionManagement>

</project>
```

---

# 🐳 9. Dockerfile


# 🚀 10. Jenkinsfile

<img width="1920" height="1080" alt="2" src="https://github.com/user-attachments/assets/71ee13f4-2fab-4dcc-894f-9cb933d3e0e2" />

# 🎯 Access Your Application

```
http://51.21.202.150:8082/currency-converter/
```
<img width="1920" height="1080" alt="1" src="https://github.com/user-attachments/assets/3a43527c-c659-45bd-be70-f63d38c3f96e" />

---
# ⚡ GITHUB ACTIONS (COMPLETE WORKING CI/CD FILE)

Create file:

```
.github/workflows/ci-cd.yml
```

**Use the YAML from .github/workflows/ci-cd-yml**


# 🔑 REQUIRED GITHUB SECRETS

Go to:

```
GitHub → Settings → Secrets and variable → Actions 
```

Add:

| Secret Name | Value |
|-------------|--------|
| `SONAR_HOST_URL` | `http://51.21.202.150:9000` |
| `SONAR_TOKEN` | SonarQube PAT |
| `NEXUS_URL` | `http://51.21.202.150:8081/repository/maven-releases/` |
| `NEXUS_USER` | admin |
| `NEXUS_PASS` | password |
| `DOCKERHUB_USERNAME` | jagadapi240 |
| `DOCKERHUB_TOKEN` | DockerHub PAT |
| `SSH_HOST` | `51.21.202.150` |
| `SSH_USER` | ubuntu |
| `SSH_KEY` | private SSH key contents |

Generate SSH key:

```
ssh-keygen -t ed25519
cat ~/.ssh/id_ed25519.pub   (add to EC2 authorized_keys)
```
<img width="1920" height="1080" alt="10" src="https://github.com/user-attachments/assets/be79d256-5ee9-44cb-be74-dabd47ca0cc9" />
<img width="1920" height="1080" alt="11" src="https://github.com/user-attachments/assets/b6309c75-e414-4d3a-9722-850fbaad5043" />



# 🔥 YOUR PIPELINES SUMMARY

### ✔️ Jenkins Pipeline  
Triggers from GitHub Webhook → builds → analyzes → deploys to Nexus → pushes Docker image → runs container on EC2.

### ✔️ GitHub Actions Pipeline  
Triggers on push → builds → analyzes → deploys to Nexus → builds Docker → pushes to DockerHub → SSH deploys to EC2.

---

# 🎉 13. Pipeline Summary

1. GitHub push triggers Jenkins  
2. Jenkins pulls code  
3. SonarQube analyses code  
4. Maven builds WAR  
5. Artifact is uploaded to Nexus  
6. Jenkins downloads WAR  
7. Docker image is built  
8. Pushed to DockerHub  
9. Container deployed on port 8082  
10. App becomes live  

---

