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
sudo apt update -y
sudo apt install -y docker.io docker-compose maven openjdk-17-jdk
sudo usermod -aG docker $USER
```

Relogin or reboot.

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
Password: <GitHub-PAT>
```

---

## 👉 DockerHub PAT (required for Jenkins push)
Generate:

1. DockerHub → Settings → Security  
2. Create Access Token  

Add in Jenkins:

- ID: `dockerhub-pat`
- Username: `jagadapi240`
- Password: `<your token>`

---

# 🏢 4. Nexus Setup (Artifact Repository)

### Run Nexus:
```bash
docker run -d --name nexus -p 8081:8081 sonatype/nexus3
```

Open Nexus:
```
http://YOUR-IP:8081/
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

# 🔍 5. SonarQube Setup

### Run SonarQube:
```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts
```

Open:
```
http://YOUR-IP:9000/
```

Login:
```
username: admin
password: admin
```

Generate token:
- My Account → Security → Generate Token

Add in Jenkins as:
- ID: `sonar-token`

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
URL: http://localhost:9000/
Token: <saved token>
```

#### Credentials Added:
- `nexus-creds` → Nexus username/password
- `dockerhub-pat` → DockerHub username/token
- `sonar-token` → SonarQube token

---

# 🌐 7. GitHub Webhook Setup

GitHub repo → Settings → Webhooks → Add Webhook:

```
Payload URL: http://YOUR-IP:8080/github-webhook/
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

```dockerfile
FROM tomcat:10.1-jdk17

RUN rm -rf /usr/local/tomcat/webapps/*

COPY deploy/app.war /usr/local/tomcat/webapps

EXPOSE 8080

CMD ["catalina.sh", "run"]
```

---

# 🚀 10. Full Jenkinsfile (Complete CI/CD Pipeline)

```groovy
pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    environment {
        SONARQUBE_ENV = 'MySonarQube'
        DOCKER_IMAGE = 'jagadapi240/currency-converter'
        VERSION = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('SCM Checkout') {
            steps { checkout scm }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv(env.SONARQUBE_ENV) {
                    sh '''
                        mvn clean verify sonar:sonar \
                          -Dsonar.projectKey=currency-converter \
                          -Dsonar.projectName="Currency Converter"
                    '''
                }
            }
        }

        stage('Maven Package') {
            steps { sh 'mvn -B -DskipTests clean package' }
        }

        stage('Nexus Upload') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                        mvn deploy -DskipTests \
                          -DaltDeploymentRepository=nexus::default::http://$NEXUS_USER:$NEXUS_PASS@51.21.202.150:8081/repository/maven-releases/
                    '''
                }
            }
        }

        stage('Download WAR from Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                        rm -rf deploy || true
                        mkdir deploy

                        curl -f -u $NEXUS_USER:$NEXUS_PASS \
                          -o deploy/app.war \
                          http://51.21.202.150:8081/repository/maven-releases/com/ajacs/currency-converter-web/1.0.1/currency-converter-web-1.0.1.war
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${DOCKER_IMAGE}:${VERSION} .
                '''
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-pat',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_TOKEN'
                )]) {
                    sh '''
                        echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}:${VERSION}
                        docker tag ${DOCKER_IMAGE}:${VERSION} ${DOCKER_IMAGE}:latest
                        docker push ${DOCKER_IMAGE}:latest
                    '''
                }
            }
        }

        stage('Deploy on Tomcat Docker') {
            steps {
                sh '''
                    PORT_CONTAINER=$(docker ps -q --filter "publish=8082")
                    if [ ! -z "$PORT_CONTAINER" ]; then
                        docker rm -f $PORT_CONTAINER
                    fi

                    docker rm -f currency-converter-app || true

                    docker run -d \
                      --name currency-converter-app \
                      -p 8082:8080 \
                      ${DOCKER_IMAGE}:${VERSION}
                '''
            }
        }
    }

    post { always { cleanWs() } }
}
```

---

# 🌍 11. Access Your Application

```
http://51.21.202.150:8082/
```

---

# 🧪 12. Troubleshooting

### Check port:
```bash
sudo lsof -i :8082
```

### Remove container:
```bash
docker rm -f <container-id>
```

### Test new image manually:
```bash
docker run -d -p 8082:8080 jagadapi240/currency-converter:latest
```

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

# ✔ Completed Successfully
This README is the **final complete documentation** for your end-to-end CI/CD implementation.

# currency-coverter-app
