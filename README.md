# 🚀 Spring PetClinic + Quote Generator Deployment using Docker & Jenkins

This project demonstrates **end-to-end DevOps automation** for deploying two applications — **Spring PetClinic** and a **Quote Generator** — on a **single AWS EC2 instance**, using **Docker**, with **Jenkins installed locally** to automate the pipeline.

---

## 🛠️ Tech Stack

| Tool             | Purpose                             |
| ---------------- | ----------------------------------- |
| Jenkins (local)  | CI/CD pipeline runner               |
| AWS EC2          | Hosting Dockerized apps             |
| Docker           | Containerizing Spring Boot apps     |
| Git & GitHub     | Version control and source code     |
| Nginx (optional) | Reverse proxy (for multi-port apps) |

---

## 📂 Repository Structure

```
.
├── Dockerfile (Spring PetClinic)
├── docker-compose.yml (Optional if managing multiple containers)
├── Jenkinsfile
├── README.md
└── scripts/
    └── deploy.sh (Optional: SSH-based deploy script)
```

---

## 🌍 Application URLs

| Application      | Port | URL Format                  |
| ---------------- | ---- | --------------------------- |
| Spring PetClinic | 8081 | http\://<your-ec2-dns>:8081 |
| Quote Generator  | 8082 | http\://<your-ec2-dns>:8082 |

---

## 🧱 Dockerfile (Spring PetClinic)

```Dockerfile
# Stage 1: Build with Maven
FROM maven:3.9.6-eclipse-temurin-17 as builder
WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests

# Stage 2: Run JAR
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8081
ENTRYPOINT ["java", "-jar", "app.jar"]
```
---

## ✅ Jenkinsfile (CI/CD Pipeline)

```groovy
pipeline {
    agent any

    environment {
        EC2_USER   = "ec2-user"
        EC2_HOST   = "ec2-54-91-8-37.compute-1.amazonaws.com"
        PEM_FILE   = "/Users/princemittal/Downloads/princemittal2.pem"
        REPO_URL   = "https://github.com/princemittal112/spring-petclinic.git"
        APP_DIR    = "/home/ec2-user/app"
        IMAGE_NAME = "spring-petclinic"
        PORT       = "8081"
    }

    stages {
        stage('Deploy to EC2') {
            steps {
                script {
                    sh """
                    ssh -o StrictHostKeyChecking=no -i ${PEM_FILE} ${EC2_USER}@${EC2_HOST} '
                        sudo yum install -y git docker || true
                        sudo systemctl start docker
                        sudo systemctl enable docker

                        # Pull latest code
                        if [ ! -d "${APP_DIR}" ]; then
                            git clone "${REPO_URL}" "${APP_DIR}"
                        else
                            cd "${APP_DIR}" && git pull
                        fi

                        cd "${APP_DIR}"

                        docker rm -f "${IMAGE_NAME}" || true
                        docker system prune -af || true

                        # Build correct Java image
                        docker build --no-cache -t "${IMAGE_NAME}" .

                        # Run container (Java app)
                        docker run -d --name "${IMAGE_NAME}" -p ${PORT}:8081 "${IMAGE_NAME}"
                    '
                    """
                }
            }
        }
    }
}


```

---

## ✅ Jenkinsfile for both websites if agent node present

```groovy
pipeline {
    agent any

    environment {
        EC2_DNS = "ec2-18-207-1-87.compute-1.amazonaws.com"
        EC2_USER = "ec2-user"
        PEM_PATH = "/Users/princemittal/Downloads/princemittal.pem"
    }

    stages {
        stage('Deploy on EC2') {
            steps {
                sh """
                ssh -tt -o StrictHostKeyChecking=no -i ${PEM_PATH} ${EC2_USER}@${EC2_DNS} << 'ENDSSH'
                    sudo yum update -y
                    sudo service docker start

                    # Spring PetClinic
                    rm -rf spring-petclinic
                    git clone https://github.com/princemittal112/spring-petclinic.git
                    cd spring-petclinic
                    docker build -t petclinic-app .
                    docker stop petclinic-container || true
                    docker rm petclinic-container || true
                    docker run -d -p 8081:8080 --name petclinic-container petclinic-app
                    cd ..

                    # Quote Generator
                    rm -rf quote-generator
                    git clone https://github.com/princemittal112/quote-generator.git
                    cd quote-generator
                    docker build -t quote-app .
                    docker stop quote-container || true
                    docker rm quote-container || true
                    docker run -d -p 8082:80 --name quote-container quote-app
                ENDSSH
                """
            }
        }
    }
}

```

> ✅ **NOTE**: Replace `your-ec2-ip`, `your-key.pem`, and local paths accordingly.

---

## ⚙️ EC2 Setup Guide

1. **Launch EC2**

   * OS: Amazon Linux 2
   * Enable port **8081** and **8082** in the Security Group

2. **Install Docker on EC2**

```bash
sudo yum update -y
sudo yum install docker -y
sudo service docker start
sudo usermod -aG docker ec2-user
```

3. **Install Java (optional)**
   For Jenkins agent/slave setup if required:

```bash
sudo amazon-linux-extras enable corretto8
sudo yum install java-1.8.0-amazon-corretto -y
```

---

## 🐳 Running Locally (Optional Debugging)

```bash
git clone https://github.com/princemittal112/spring-petclinic.git
cd spring-petclinic
docker build -t petclinic-app .
docker run -d -p 8081:8081 petclinic-app
```

---

## 🧠 Key Learnings

* SSH errors during Jenkins remote pipeline were fixed by using **EC2 DNS instead of IP**
* Deployed **2 apps on same EC2**, running on different ports: 8081 (Spring PetClinic), 8082 (Quote Generator)
* **Jenkins runs locally**, while all Docker deployment happens on **EC2**
* Learned to split Docker build stages for optimal image size
* Used Jenkins pipeline with **inline SSH block** instead of agent/slave nodes due to persistent SSH failures

---

## 🔥 Troubleshooting Notes

| Issue                                | Fix                                                              |
| ------------------------------------ | ---------------------------------------------------------------- |
| `docker: command not found`          | Docker not installed on EC2 or not in \$PATH                     |
| Jenkins node "launch via SSH" failed | Used SSH in pipeline directly instead of configuring agent node  |
| `Connection reset by peer`           | Switched to EC2 DNS instead of public IP                         |
| `Authentication failed` in Jenkins   | Uploaded correct private key `.pem` to Jenkins Credentials Store |

<img width="1440" height="900" alt="Screenshot 2025-07-11 at 12 13 31 PM" src="https://github.com/user-attachments/assets/4cf2abb2-3da6-41d6-a005-2f38a8a46ec8" />

<img width="1440" height="900" alt="Screenshot 2025-07-11 at 11 58 37 AM" src="https://github.com/user-attachments/assets/c01843d4-0ead-4ebe-b04e-88f1b5a74059" />


