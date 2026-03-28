# EasyCRUD — Jenkins CI/CD Pipeline on AWS

> Full-stack Student Registration application (Spring Boot + React) deployed on AWS using Jenkins pipelines, Docker containers, S3 artifact storage, and RDS MySQL.

**Recommended Repository Name:** `easycrud-jenkins-cicd-aws`

---

## Table of Contents

1. [Tech Stack](#tech-stack)
2. [Architecture](#architecture)
3. [Pre-Flight: Correct Setup Order](#pre-flight-correct-setup-order)
4. [Step 1 — Security Groups](#step-1--security-groups)
5. [Step 2 — IAM Role](#step-2--iam-role)
6. [Step 3 — RDS MySQL](#step-3--rds-mysql)
7. [Step 4 — S3 Bucket](#step-4--s3-bucket)
8. [Step 5 — Launch EC2 Instances](#step-5--launch-ec2-instances)
9. [Step 6 — Backend EC2 Setup](#step-6--backend-ec2-setup)
10. [Step 7 — Frontend EC2 Setup](#step-7--frontend-ec2-setup)
11. [Step 8 — Jenkins EC2 Setup](#step-8--jenkins-ec2-setup)
12. [Step 9 — Repository Configuration](#step-9--repository-configuration)
13. [Step 10 — Jenkins Plugins](#step-10--jenkins-plugins)
14. [Step 11 — Jenkins Tools Configuration](#step-11--jenkins-tools-configuration)
15. [Step 12 — Jenkins Credentials](#step-12--jenkins-credentials)
16. [Step 13 — S3 Artifact Storage Configuration](#step-13--s3-artifact-storage-configuration)
17. [Step 14 — The Three Pipelines](#step-14--the-three-pipelines)
18. [Step 15 — Run and Verify](#step-15--run-and-verify)
19. [Troubleshooting](#troubleshooting)
20. [Project Structure](#project-structure)
21. [Glossary](#glossary)
22. [Final Checklist](#final-checklist)

---

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Backend | Spring Boot 3.3 · Java 17 | REST API — Student CRUD operations |
| Frontend | React 18 | User interface — served via Docker + Nginx |
| CI/CD | Jenkins | Pipeline orchestration — build and deploy |
| Containers | Docker | Packages apps for consistent deployment |
| Artifact Store | AWS S3 | Stores compiled JAR between pipeline jobs |
| Database | AWS RDS MySQL 8.0 | Managed relational DB — student records |
| Build Tool | Apache Maven | Compiles and packages Spring Boot |
| Node Runtime | Node.js 20 (on Jenkins) | Builds the React frontend |

---

## Architecture

```
                   ┌────────────────────────────────────┐
                   │         GitHub Repository           │
                   │   github.com/Shefalihub/EasyCurd    │
                   └─────────────┬──────────────────────┘
                                 │  git pull
                   ┌─────────────▼──────────────────────┐
                   │       Jenkins EC2  (t2.medium)      │
                   │  ┌────────┐ ┌────────┐ ┌─────────┐ │
                   │  │ Job 1  │ │ Job 2  │ │  Job 3  │ │
                   │  │ Maven  │ │  SSH   │ │ Docker  │ │
                   │  │ Build  │ │ Deploy │ │  Build  │ │
                   │  └───┬────┘ └───┬────┘ └────┬────┘ │
                   └──────┼──────────┼────────────┼──────┘
                          │          │            │
                      JAR to S3   SSH +       docker save
                          │      Docker          + scp
                   ┌──────▼───┐ ┌───▼──────┐ ┌──▼──────────┐
                   │    S3    │ │ Backend  │ │  Frontend   │
                   │  Bucket  │ │   EC2    │ │    EC2      │
                   │  (JAR)   │ │  :8081   │ │    :80      │
                   └──────────┘ └────┬─────┘ └─────────────┘
                                     │ JDBC
                              ┌──────▼──────┐
                              │ RDS MySQL   │
                              │  Private    │
                              │  Subnet     │
                              └─────────────┘
```

---

## Pre-Flight: Correct Setup Order

> Follow this exact sequence. Creating resources in the wrong order means going back and reassigning things.

| # | Task | Why This Order |
|---|---|---|
| 1 | Create Security Groups (all 4) | EC2 launch requires an SG — do this first |
| 2 | Create IAM Role | Attach to EC2 at launch time — easier than after |
| 3 | Create RDS | Note the endpoint before writing `application.properties` |
| 4 | Create S3 Bucket | Must exist before Job1 runs |
| 5 | Launch EC2 Instances | Attach SG + IAM role at launch |
| 6 | Set up Backend EC2 | Install Docker, AWS CLI, create DB schema |
| 7 | Set up Frontend EC2 | Install Docker |
| 8 | Set up Jenkins EC2 | Install Jenkins, Maven, Docker |
| 9 | Configure repo files | Now you have the RDS endpoint and EC2 IPs |
| 10 | Jenkins plugins, tools, credentials | Server must be running first |
| 11 | Create and run pipelines | Everything else must be ready |

---

## Step 1 — Security Groups

Go to: **AWS Console → EC2 → Security Groups → Create Security Group**

Create all four Security Groups before launching any EC2 or RDS.

### Jenkins SG — `jenkins-sg`

| Type | Port | Source | Purpose |
|---|---|---|---|
| SSH | 22 | Your IP | SSH access to manage server |
| Custom TCP | 8080 | Your IP | Jenkins web UI |

### Backend SG — `backend-sg`

| Type | Port | Source | Purpose |
|---|---|---|---|
| SSH | 22 | `jenkins-sg` (select the SG, not an IP) | Jenkins deploys via SSH |
| Custom TCP | 8081 | 0.0.0.0/0 | Public API access |

### Frontend SG — `frontend-sg`

| Type | Port | Source | Purpose |
|---|---|---|---|
| SSH | 22 | `jenkins-sg` (select the SG, not an IP) | Jenkins deploys via SSH |
| HTTP | 80 | 0.0.0.0/0 | Public web access |

### RDS SG — `rds-sg`

| Type | Port | Source | Purpose |
|---|---|---|---|
| MySQL/Aurora | 3306 | `backend-sg` (select the SG) | Only Backend EC2 can reach DB |

> ⚠️ **Never expose port 3306 to 0.0.0.0/0.** Restricting MySQL to the Backend Security Group only is mandatory. Exposing it to the internet is a critical security risk.

---

## Step 2 — IAM Role

An IAM Role attached to EC2 grants AWS permissions without any hardcoded access keys. Both Jenkins EC2 and Backend EC2 need S3 access.

1. Go to **AWS Console → IAM → Roles → Create Role**
2. Trusted entity type: **AWS Service → EC2**
3. Attach permission policy: `AmazonS3FullAccess`
4. Role name: `jenkins-s3-role`
5. Click **Create Role**

> ℹ️ You will attach this role when launching EC2 instances in Step 5. It can also be attached after launch via **EC2 → Actions → Security → Modify IAM Role**.

---

## Step 3 — RDS MySQL

### Create the Database Instance

1. **AWS Console → RDS → Create Database**
2. Engine: **MySQL 8.0**
3. Template: **Free Tier**
4. DB instance identifier: `database-1`
5. Master username: `admin`
6. Master password: `admin123`
7. DB instance class: `db.t3.micro`
8. VPC: same VPC as your EC2 instances
9. **Public access: `No`** ← keep this No
10. VPC Security Group: select `rds-sg`
11. Initial database name: `student_db`
12. Click **Create Database** — wait ~5 minutes
13. Note the **Endpoint** shown after creation (e.g. `database-1.xxxx.us-east-1.rds.amazonaws.com`)

> ⚠️ **Public access must be No.** Setting it to Yes exposes your database endpoint to the internet. The Backend EC2 communicates with RDS privately within the VPC.

### Fix: SSL Requirement Error

AWS RDS MySQL 8.0 enforces SSL by default. If you see this error after deployment:

```
Connections using insecure transport are prohibited while --require_secure_transport=ON
```

**Fix — disable SSL in RDS Parameter Group:**

1. **RDS → Parameter Groups → Create parameter group**
2. Family: `mysql8.0` → Name: `my-params` → Create
3. Edit → search `require_secure_transport` → set value to `OFF`
4. **RDS → Your database → Modify** → Parameter group: `my-params` → Apply immediately
5. Reboot the RDS instance

### Create the Database Schema

Connect from your Backend EC2 after RDS is available:

```bash
# Install MySQL client
apt install -y mysql-client

# Connect to RDS
mysql -h YOUR_RDS_ENDPOINT -u admin -p

# Inside MySQL:
CREATE DATABASE student_db;
SHOW DATABASES;
EXIT;
```

---

## Step 4 — S3 Bucket

Job1 uploads the compiled JAR to S3. Job2 pulls it from S3 to the Backend EC2.

1. **AWS Console → S3 → Create Bucket**
2. Bucket name: `project-ultron-007` (must be globally unique — choose your own)
3. Region: `us-east-1` (same as your EC2 instances)
4. Object Ownership: **ACLs enabled**
5. Keep Block Public Access settings (artifacts are accessed via IAM role, not public URLs)
6. Click **Create Bucket**

> ℹ️ Note your bucket name — you will use it in the Job1 and Job2 pipeline scripts.

---

## Step 5 — Launch EC2 Instances

Use **Ubuntu 22.04 LTS** for all three instances.

| Instance | Type | Security Group | IAM Role | Key Pair |
|---|---|---|---|---|
| Jenkins EC2 | t2.medium | `jenkins-sg` | `jenkins-s3-role` | your-keypair.pem |
| Backend EC2 | t2.micro | `backend-sg` | `jenkins-s3-role` | your-keypair.pem |
| Frontend EC2 | t2.micro | `frontend-sg` | None required | your-keypair.pem |

> ℹ️ Use the same key pair for all three instances. Jenkins SSHes into Backend and Frontend EC2 using this key pair. You will paste its contents into Jenkins credentials later.

---

## Step 6 — Backend EC2 Setup

SSH into Backend EC2 and run all commands below. This is a one-time setup.

```bash
# Connect
ssh -i your-keypair.pem ubuntu@YOUR_BACKEND_EC2_PUBLIC_IP

# Update packages
apt update -y

# Install Docker
apt install docker.io -y
systemctl enable --now docker

# Add ubuntu user to docker group
usermod -aG docker ubuntu

# Install AWS CLI (to pull JAR from S3)
apt install -y awscli
snap install aws-cli --classic

# Verify S3 access (works via IAM role — no keys needed)
aws s3 ls

# Login to DockerHub
docker login -u YOUR_DOCKERHUB_USERNAME

# Install MySQL client (to verify RDS connection)
apt install -y mysql-client

# Create the database schema on RDS
mysql -h YOUR_RDS_ENDPOINT -u admin -p
# Inside MySQL:
# CREATE DATABASE student_db;
# SHOW DATABASES;
# EXIT;

# Log out and back in for docker group to take effect
exit
```

---

## Step 7 — Frontend EC2 Setup

SSH into Frontend EC2. Only Docker is needed — the React app runs inside a container.

```bash
# Connect
ssh -i your-keypair.pem ubuntu@YOUR_FRONTEND_EC2_PUBLIC_IP

# Update packages
apt update -y

# Install Docker
apt install docker.io -y
systemctl enable --now docker

# Add ubuntu user to docker group
usermod -aG docker ubuntu

# Log out and back in for docker group to take effect
exit
```

> ℹ️ No Node.js, npm, or Nginx needed on Frontend EC2. The React app is built on Jenkins EC2, packaged into a Docker image, and shipped as a `.tar` file to this instance.

---

## Step 8 — Jenkins EC2 Setup

SSH into Jenkins EC2 and run all commands in order.

```bash
# Connect
ssh -i your-keypair.pem ubuntu@YOUR_JENKINS_EC2_PUBLIC_IP

# Update packages
apt update -y

# Install Java 17 (required by Jenkins)
apt install openjdk-17-jdk -y

# Add Jenkins apt repository
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins
sudo apt update
sudo apt install jenkins -y

# Install Maven (for Spring Boot build)
apt install maven -y

# Install AWS CLI
snap install aws-cli --classic
aws configure
# Enter: Access Key, Secret Key, region: us-east-1, output: json

# Install Docker (Jenkins builds Docker images)
apt install docker.io -y
systemctl enable --now docker

# Add jenkins user to docker group — CRITICAL
sudo usermod -aG docker jenkins

# Restart Jenkins to apply docker group membership
systemctl restart jenkins

# Verify jenkins user can run docker (must show empty table, not permission error)
sudo -u jenkins docker ps
```

### Get Initial Admin Password

```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```

Access Jenkins at: `http://YOUR_JENKINS_EC2_PUBLIC_IP:8080`

> ⚠️ If `sudo -u jenkins docker ps` shows a permission error, the group change did not apply. Run `systemctl restart jenkins` again and retry.

---

## Step 9 — Repository Configuration

Update these files in your repository before running any pipeline.

### `backend/src/main/resources/application.properties`

```properties
spring.datasource.url=jdbc:mariadb://YOUR_RDS_ENDPOINT:3306/student_db
spring.datasource.username=admin
spring.datasource.password=admin123
spring.datasource.driver-class-name=org.mariadb.jdbc.Driver
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
server.port=8081
```

> ⚠️ Keep `server.port=8081`. Jenkins runs on port 8080 on the Jenkins EC2. The backend runs on the Backend EC2 on port 8081. Using 8081 avoids conflict and confusion.

> ⚠️ Use `jdbc:mariadb://` not `jdbc:mysql://`. This project uses the MariaDB JDBC driver. Using `mysql://` causes a driver class not found error at startup.

### `frontend/.env`

Create this file if it does not exist:

```env
REACT_APP_API_URL=http://YOUR_BACKEND_EC2_PUBLIC_IP:8081
```

### `frontend/Dockerfile`

Create this file if it does not exist:

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## Step 10 — Jenkins Plugins

Go to: **Manage Jenkins → Plugins → Available Plugins**

Search and install each plugin:

| Plugin | Purpose |
|---|---|
| Pipeline | Enables declarative Jenkinsfile pipelines |
| Git | Pulls source code from GitHub |
| SSH Agent | Allows Jenkins to SSH into Backend and Frontend EC2 |
| Maven Integration | Enables `mvn` commands and Maven tool configuration |
| Artifact Manager on S3 | Stores build artifacts directly in S3 bucket |
| AWS Credentials | Stores AWS access keys securely in Jenkins |
| Credentials Binding | Injects stored credentials as environment variables |
| NodeJS | Installs and manages Node.js for React builds |
| Pipeline Stage View | Visual progress view for pipeline stages (optional) |

> ℹ️ After installing all plugins, Jenkins will prompt you to restart. Do so before continuing.

---

## Step 11 — Jenkins Tools Configuration

Go to: **Manage Jenkins → Tools**

### JDK
- Click **Add JDK**
- Name: `jdk-17`
- Uncheck "Install automatically"
- JAVA_HOME: `/usr/lib/jvm/java-17-openjdk-amd64`

### Maven
- Click **Add Maven**
- Name: `maven`
- Check "Install automatically" → version `3.9.x`

### NodeJS
- Click **Add NodeJS**
- Name: `node20`
- Check "Install automatically" → version `20.x`

---

## Step 12 — Jenkins Credentials

Go to: **Manage Jenkins → Credentials → (global) → Add Credentials**

### SSH Key — for EC2 access

| Field | Value |
|---|---|
| Kind | SSH Username with private key |
| Scope | Global |
| ID | `key` ← this exact ID is used in all three pipelines |
| Username | `ubuntu` |
| Private Key | Enter directly → paste your `.pem` file contents |

### AWS Credentials — if not using IAM Role

| Field | Value |
|---|---|
| Kind | AWS Credentials |
| ID | `aws-creds` |
| Access Key ID | your-access-key |
| Secret Access Key | your-secret-key |

> ✅ If you attached the `jenkins-s3-role` IAM role to Jenkins EC2 in Step 2, AWS CLI commands in your pipeline work automatically — no AWS credentials entry needed in Jenkins.

---

## Step 13 — S3 Artifact Storage Configuration

Connect Jenkins to your S3 bucket so `archiveArtifacts` in Job1 uploads to S3.

1. Go to: **Manage Jenkins → System**
2. Scroll to: **Artifact Management for Builds**
3. Provider: **Amazon S3**
4. S3 Bucket: `project-ultron-007`
5. Region: `us-east-1`
6. Credentials: select `aws-creds` (or leave blank if using IAM role)
7. Click **Save**

---

## Step 14 — The Three Pipelines

Create three Pipeline jobs in Jenkins: **New Item → Pipeline**

### Job 1 — S3 Pipeline (Build JAR + Upload to S3)

Pulls from GitHub, builds the Spring Boot JAR using Maven, archives to S3.

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk-17'
        maven 'maven'
    }
    stages {
        stage('Pull') {
            steps {
                git branch: 'main', url: 'https://github.com/Shefalihub/EasyCurd.git'
            }
        }
        stage('Build') {
            steps {
                sh '''
                    cd backend
                    mvn clean package -DskipTests=true
                '''
            }
        }
        stage('Push to S3') {
            steps {
                archiveArtifacts artifacts: 'backend/target/*.jar', followSymlinks: false
            }
        }
    }
}
```

---

### Job 2 — Backend Deploy (SSH + Docker)

SSHes into Backend EC2, pulls the JAR from S3, builds a Docker image, runs container on port 8081.

> Before running: update `EC2_IP` with your Backend EC2 public IP, and `S3_PATH` with the correct Job1 build number. Check your S3 bucket path in the AWS console after Job1 runs.

```groovy
pipeline {
    agent any

    environment {
        EC2_IP  = "YOUR_BACKEND_EC2_PUBLIC_IP"
        S3_PATH = "s3://project-ultron-007/job1/BUILD_NUMBER/artifacts/backend/target/student-registration-backend-0.0.1-SNAPSHOT.jar"
        IMAGE   = "YOUR_DOCKERHUB_USERNAME/myjenkins:v1"
    }

    stages {
        stage('Deploy to Backend EC2') {
            steps {
                sshagent(['key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${EC2_IP} '
                        set -e
                        echo "Stopping old container..."
                        docker stop my-app || true
                        docker rm my-app || true

                        echo "Cleaning old files..."
                        rm -f app.jar Dockerfile

                        echo "Downloading artifact from S3..."
                        aws s3 cp ${S3_PATH} app.jar

                        echo "Creating Dockerfile..."
                        cat > Dockerfile << "DOCKERFILEEOF"
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY app.jar .
EXPOSE 8081
CMD ["java", "-jar", "app.jar"]
DOCKERFILEEOF

                        echo "Building Docker image..."
                        docker build -t ${IMAGE} .

                        echo "Running container..."
                        docker run -d -p 8081:8081 --name my-app ${IMAGE}

                        echo "Deployment successful!"
                    '
                    """
                }
            }
        }
    }
}
```

---

### Job 3 — Frontend Deploy (Docker + SCP)

Builds a React Docker image on Jenkins EC2, saves it as a `.tar`, copies to Frontend EC2, runs on port 80.

```groovy
pipeline {
    agent any

    environment {
        EC2_IP = "YOUR_FRONTEND_EC2_PUBLIC_IP"
        IMAGE  = "YOUR_DOCKERHUB_USERNAME/myapp:frontend1"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Shefalihub/EasyCurd.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    cd frontend
                    docker build -t YOUR_DOCKERHUB_USERNAME/myapp:frontend1 .
                '''
            }
        }

        stage('Deploy to Frontend EC2') {
            steps {
                sshagent(['key']) {
                    sh """
                        docker save $IMAGE > frontend.tar

                        scp -o StrictHostKeyChecking=no frontend.tar ubuntu@$EC2_IP:/home/ubuntu/

                        ssh -o StrictHostKeyChecking=no ubuntu@$EC2_IP '
                            docker stop frontend || true
                            docker rm frontend || true
                            docker load < frontend.tar
                            docker run -d -p 80:80 --name frontend $IMAGE
                        '
                    """
                }
            }
        }
    }
}
```

---

## Step 15 — Run and Verify

### Run Order

> Always run in this exact order — Job2 depends on an artifact produced by Job1.

| Order | Job | Verify After |
|---|---|---|
| 1st | Job1 (S3) | Check S3 bucket — `.jar` file appears under `job1/1/artifacts/...` |
| 2nd | Job2 (Backend) | `docker ps` on Backend EC2 — `my-app` shows as `Up` on port 8081 |
| 3rd | Job3 (Frontend) | `docker ps` on Frontend EC2 — `frontend` shows as `Up` on port 80 |

### Verify Backend

```bash
# On Backend EC2
docker ps
# Expected: my-app   Up X minutes   0.0.0.0:8081->8081/tcp

# Test API
curl http://localhost:8081/students
```

### Verify Frontend

```bash
# On Frontend EC2
docker ps
# Expected: frontend   Up X minutes   0.0.0.0:80->80/tcp
```

### Verify Data in RDS

After registering a student through the UI:

```bash
mysql -h YOUR_RDS_ENDPOINT -u admin -p
USE student_db;
SELECT * FROM student;
# Your registered student record appears here
```

### Open in Browser

| Service | URL |
|---|---|
| Frontend | `http://YOUR_FRONTEND_EC2_PUBLIC_IP` |
| Backend API | `http://YOUR_BACKEND_EC2_PUBLIC_IP:8081` |

---

## Troubleshooting

### ❌ `docker: not found` — Exit Code 127

```
docker: not found
script.sh.copy: 3: docker: not found
```

**Cause:** Docker not installed on Jenkins EC2, or `jenkins` user not in the docker group.

```bash
apt install docker.io -y
systemctl enable --now docker
sudo usermod -aG docker jenkins
systemctl restart jenkins

# Verify — must show empty table, not an error
sudo -u jenkins docker ps
```

---

### ❌ `Unknown database 'student_db'`

```
Unknown database 'student_db'
SQLSyntaxErrorException: (conn=64) Unknown database 'student_db'
```

**Cause:** The database schema was never created on RDS. RDS creates the instance but not the schema inside it.

```bash
mysql -h YOUR_RDS_ENDPOINT -u admin -p
CREATE DATABASE student_db;
EXIT;

# Then restart the container
docker stop my-app && docker rm my-app
docker run -d -p 8081:8081 --name my-app ... (same run command as before)
```

---

### ❌ `Failed to load driver class com.mysql.cj.jdbc.Driver`

```
Failed to load driver class com.mysql.cj.jdbc.Driver
in either of HikariConfig class loader or Thread context classloader
```

**Cause:** The JAR was built with the MariaDB JDBC driver but the JDBC URL uses `mysql://` protocol, which expects the MySQL Connector/J driver class.

```bash
# Wrong
SPRING_DATASOURCE_URL=jdbc:mysql://YOUR_RDS_ENDPOINT:3306/student_db

# Correct
SPRING_DATASOURCE_URL=jdbc:mariadb://YOUR_RDS_ENDPOINT:3306/student_db
```

---

### ❌ `Connections using insecure transport are prohibited`

```
Connections using insecure transport are prohibited
while --require_secure_transport=ON
```

**Cause:** AWS RDS MySQL 8.0 enforces SSL by default.

**Fix A — Disable SSL on RDS (recommended for dev/learning):**

1. RDS → Parameter Groups → Edit → `require_secure_transport` → `OFF`
2. Reboot RDS instance

**Fix B — Enable SSL in the JDBC URL (no RDS change needed):**

```bash
# Wrap in quotes — & is a special character in bash
docker run -d -p 8081:8081 --name my-app \
  -e "SPRING_DATASOURCE_URL=jdbc:mariadb://ENDPOINT:3306/student_db?useSSL=true&trustServerCertificate=true" \
  -e SPRING_DATASOURCE_USERNAME=admin \
  -e SPRING_DATASOURCE_PASSWORD=admin123 \
  IMAGE_NAME
```

---

### ❌ `usermod: user 'jenkins' does not exist`

```
usermod: user 'jenkins' does not exist
```

**Cause:** You ran the `usermod` command on the wrong EC2. Jenkins only exists on the Jenkins EC2.

```bash
# Verify you are on the right machine
systemctl status jenkins   # should show Active: running
```

---

### ❌ `docker: invalid reference format`

```
docker: invalid reference format
```

**Cause:** Pasting a multi-line docker run command with backslash continuations into the terminal breaks the image name parsing.

```bash
# Wrong — multi-line paste with backslashes breaks
docker run -d \
  -e VAR=value \
  image-name

# Correct — single line, no backslashes
docker run -d -e VAR=value image-name
```

---

### ❌ Container exits immediately — Exit Code 1

```
Exited (1) 30 seconds ago
```

**Cause:** Application crashed on startup. Always check logs first:

```bash
docker logs my-app
```

Common causes found in logs: Unknown database, wrong JDBC driver, SSL error, missing environment variables.

---

### ❌ Missing `SPRING_DATASOURCE_USERNAME` or `PASSWORD`

**Cause:** The `docker run` command was missing the username and/or password `-e` flags.

**Diagnose:**

```bash
docker inspect my-app | grep -A 20 "Env"
# If USERNAME or PASSWORD are missing from the list, that is the cause
```

**Fix — run as a single line with all three `-e` flags:**

```bash
docker run -d -p 8081:8081 --name my-app -e SPRING_DATASOURCE_URL=jdbc:mariadb://ENDPOINT:3306/student_db -e SPRING_DATASOURCE_USERNAME=admin -e SPRING_DATASOURCE_PASSWORD=admin123 IMAGE_NAME
```

---

## Project Structure

```
easycrud-jenkins-cicd-aws/
├── backend/
│   ├── src/
│   │   └── main/
│   │       └── resources/
│   │           └── application.properties   ← add RDS endpoint here
│   └── pom.xml
├── frontend/
│   ├── src/
│   ├── .env                                 ← add backend EC2 IP here
│   ├── Dockerfile                           ← required for Job3
│   └── package.json
└── README.md
```

---

## Glossary

| Term | What It Means in This Project |
|---|---|
| Jenkins | Automation server that runs build and deploy steps |
| Pipeline | A Groovy script defining stages: pull → build → deploy |
| Docker | Packages app and dependencies into a portable container |
| Docker image | A blueprint — running it creates a container |
| Docker container | A running instance of an image — isolated process on EC2 |
| S3 | AWS object storage — passes the JAR between Job1 and Job2 |
| RDS | AWS managed MySQL — stores student data, no DB admin needed |
| IAM Role | AWS permission attached to EC2 — grants S3 access without hardcoded keys |
| Security Group | AWS firewall — controls which ports accept traffic from where |
| SSH Agent | Jenkins plugin — SSHes into other EC2s using stored key credentials |
| `sshagent(['key'])` | Injects the `key` SSH credential so Jenkins can SSH securely |
| `archiveArtifacts` | Jenkins built-in — saves build outputs, configured to push to S3 |
| JDBC URL | Connection string telling Spring Boot how to reach the database |
| MariaDB driver | The JDBC driver bundled in this project — compatible with MySQL 8.0 on RDS |
| `docker save` / `load` | Exports a Docker image to a `.tar` file and imports it on another machine |
| VPC | Virtual Private Cloud — the private AWS network where all resources live |

---

## Final Checklist

Run through this before testing the application in the browser:

- [ ] All four Security Groups created with correct inbound rules
- [ ] IAM role `jenkins-s3-role` attached to Jenkins EC2 and Backend EC2
- [ ] RDS is available and `student_db` database created inside it
- [ ] S3 bucket exists with ACLs enabled
- [ ] Backend EC2: Docker running, ubuntu in docker group, `docker login` done
- [ ] Frontend EC2: Docker running, ubuntu in docker group
- [ ] Jenkins EC2: `sudo -u jenkins docker ps` shows empty table (no error)
- [ ] `application.properties` updated with RDS endpoint and `jdbc:mariadb://`
- [ ] `frontend/.env` updated with Backend EC2 public IP
- [ ] `frontend/Dockerfile` exists in the repo
- [ ] All Jenkins plugins installed and Jenkins restarted
- [ ] JDK 17, Maven, NodeJS 20 configured under Jenkins Tools
- [ ] SSH credential with ID `key` added to Jenkins Credentials
- [ ] S3 artifact storage configured under Jenkins System
- [ ] Job1 ran successfully — `.jar` visible in S3 bucket
- [ ] Job2 ran successfully — `docker ps` on Backend EC2 shows `my-app` Up on port 8081
- [ ] Job3 ran successfully — `docker ps` on Frontend EC2 shows `frontend` Up on port 80
- [ ] Student registered via UI — record visible in RDS via `SELECT * FROM student`

---

*Built as a DevOps learning project demonstrating end-to-end CI/CD on AWS.*

**Stack:** Spring Boot · React · Jenkins · Docker · AWS EC2 · AWS RDS · AWS S3 · Maven
