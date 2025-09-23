# CI/CD Pipeline with Jenkins, Maven, and Nexus on Ubuntu

This project demonstrates how to set up a complete CI/CD pipeline using **Jenkins**, **Maven**, and **Nexus** on an Ubuntu server. The pipeline automates code build, artifact deployment to Nexus, and deployment to an Nginx web server.

---

## 🚀 Prerequisites

* **AWS EC2 Instance**: Ubuntu Server (T2 Medium, 30 GB storage)
* **Basic knowledge** of Linux commands, Jenkins, Maven, and Docker

---

## ⚙️ Setup Steps

### 1. Create Ubuntu Server

* Instance type: **t2.medium**
* Storage: **30 GB**

---

### 2. Install Jenkins

Follow the official Jenkins installation guide:
👉 [Jenkins Installation on Linux](https://www.jenkins.io/doc/book/installing/linux/)

---

### 3. Install Maven

```bash
sudo apt install maven -y
```

* Installed version: **Maven 3.8.7**

---

### 4. Nexus Installation on Docker

```bash
sudo apt install docker.io -y
sudo docker pull sonatype/nexus3
sudo docker volume create nexus-data
sudo docker run -d -p 8081:8081 --name nexus -v nexus-data:/nexus-data sonatype/nexus3
```

---

### 5. Fetch Nexus Admin Password

```bash
docker ps
docker exec -it nexus /bin/bash
cat /nexus-data/admin.password
```

Example output:

```
da2a59b0-3c0d-4b8e-b309-fc5d3e0e7c0c

set nexus username = admin
set nexus password = admin123
```

---

### 6. Install Jenkins Plugins

* **Pipeline Maven Integration**
  ![WhatsApp Image 2025-09-22 at 21 51 42_04b71ac6](https://github.com/user-attachments/assets/403fc6cc-f598-47bc-a963-78cbfe96d2dc)

* **Nexus Artifact Uploader**
  ![WhatsApp Image 2025-09-22 at 21 47 39_1d311dff](https://github.com/user-attachments/assets/82f7db74-71f5-40ab-abf5-85c33bb9e22a)


---

### 7. Configure Maven in Jenkins

Navigate to:
`Manage Jenkins -> Tools -> Configure Maven installation`

* **Name**: `M3`
* **MAVEN\_HOME**: `/usr/share/maven`

---

### 8. Add Global Maven Settings

Navigate to:
`Manage Jenkins -> Managed Files -> Add new config (Global Maven settings.xml)`

Add content from line no 112:

```xml
<servers>
    <server>
      <id>maven-releases</id>
      <username>admin</username>
      <password>admin123</password>
    </server>
    <server>
      <id>maven-snapshots</id>
      <username>admin</username>
      <password>admin123</password>
    </server>
</servers>
```

---

### 9. Configure Settings.xml on Jenkins Server

```bash
ls -l /var/lib/jenkins/.m2/
mkdir -p /var/lib/jenkins/.m2
mkdir -p /var/lib/jenkins/.m2/repository
chown -R jenkins:jenkins /var/lib/jenkins/.m2
sudo chmod -R 755 /var/lib/jenkins/.m2
vim /var/lib/jenkins/.m2/settings.xml
```

Add:

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                              https://maven.apache.org/xsd/settings-1.0.0.xsd">

  <servers>
    <server>
      <id>maven-releases</id>
      <username>admin</username>
      <password>admin123</password>
    </server>
    <server>
      <id>maven-snapshots</id>
      <username>admin</username>
      <password>admin123</password>
    </server>
  </servers>
</settings>
```

---

### 10. Update POM.xml

Edit `POM.xml` with server IP (example: `192.168.18.141`).

```
vim  pom.xml

line no = 32:-  <url>http://192.168.18.141:8081/repository/maven-releases/</url>
line no = 36:-  <url>http://192.168.18.141:8081/repository/maven-snapshots/</url>
line no = 206:- <url>http://192.168.18.141:8081/repository/maven-releases/</url>
line no = 210:- <url>http://192.168.18.141:8081/repository/maven-snapshots/</url>

```
---

### 11. Jenkinsfile (Pipeline Script)

```groovy
pipeline {
    agent any

    environment {
        APP_NAME = "myapp"
        MAVEN_REPO = "/tmp/maven-repo"
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/rohitrawat891997/maven-simple-master.git'
            }
        }

        stage('Build with Maven') {
            steps {
                sh "mvn -Dmaven.repo.local=${MAVEN_REPO} clean package"
            }
        }

        stage('Upload to Nexus') {
            steps {
                withMaven(maven: 'M3') {
                    sh "mvn -Dmaven.repo.local=${MAVEN_REPO} clean deploy -s /var/lib/jenkins/.m2/settings.xml -DskipTests"
                }
            }
        }

        stage('Download from Nexus') {
            steps {
                sh """
                curl -u admin:admin123 -O \
                http://192.168.18.141:8081/repository/maven-releases/com/github/jitpack/maven-simple/0.2-SNAPSHOT/maven-simple-0.2-SNAPSHOT.war
                """
            }
        }

        stage("Deploy using Dockerfile") {
            steps {
                sh """
                docker run -itd --name=tomcat -e  ALLOW_EMPTY_PASSWORD=yes -p 8082:8080 bitnami/tomcat
                docker cp webapp/target/*.war tomcat:/opt/bitnami/tomcat/webapps
                docker restart tomcat
                """
            }
        }
    }
}

```
```
### Run these commands for permission

usermod -aG docker  $USER
usermod -aG docker jenkins
newgrp
systemctl restart docker

```

---

### 12. Configure Sudo Permissions for Jenkins

Edit sudoers file:

```bash
sudo vim /etc/sudoers
```

Add:

```
jenkins ALL=(ALL) NOPASSWD: /bin/systemctl restart nginx, /bin/rm, /bin/cp
```



---

## ✅ Final Workflow

1. Jenkins pulls code from GitHub
2. Builds with Maven
3. Uploads artifacts to Nexus
4. Downloads artifact from Nexus
5. Deploys artifact to **Tomcat**

   URL:- http://localhost:8082/webapp

<img width="657" height="344" alt="image" src="https://github.com/user-attachments/assets/9df45fde-0f19-44c6-93d8-b8b658d120dd" />

---

