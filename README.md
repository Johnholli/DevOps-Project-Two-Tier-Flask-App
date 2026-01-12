# 🚀 CI/CD Two-Tier Flask Application (Jenkins + Docker)

![CI/CD](https://img.shields.io/badge/CI%2FCD-Jenkins-blue)
![Docker](https://img.shields.io/badge/Docker-Containerized-blue)
![Docker Compose](https://img.shields.io/badge/Docker--Compose-Orchestration-blue)
![Python](https://img.shields.io/badge/Python-Flask-green)
![Database](https://img.shields.io/badge/Database-MySQL%205.7-orange)

---

## 📌 Overview

This project demonstrates a **production CI/CD pipeline** that builds and deploys a **two-tier web application** on an AWS EC2 instance using **Flask** and **MySQL**, fully automated with **Jenkins**, **Docker**, and **Docker Compose**. A full CI/CD pipeline is established using Jenkins to automate the build and deployment process whenever new code is pushed to a GitHub repository.

The focus of this project is not just deployment, but **real-world CI/CD troubleshooting and hardening**, including fixing pipeline failures, container conflicts, orchestration issues, and YAML errors commonly encountered in production environments.

---

## 🧱 Architecture
```
+-----------------+      +----------------------+      +-----------------------------+
|   Developer     |----->|     GitHub Repo      |----->|        Jenkins Server       |
| (pushes code)   |      | (Source Code Mgmt)   |      |  (on AWS EC2)               |
+-----------------+      +----------------------+      |                             |
                                                       | 1. Clones Repo              |
                                                       | 2. Builds Docker Image      |
                                                       | 3. Runs Docker Compose      |
                                                       +--------------+--------------+
                                                                      |
                                                                      | Deploys
                                                                      v
                                                       +-----------------------------+
                                                       |      Application Server     |
                                                       |      (Same AWS EC2)         |
                                                       |                             |
                                                       | +-------------------------+ |
                                                       | | Docker Container: Flask | |
                                                       | +-------------------------+ |
                                                       |              |              |
                                                       |              v              |
                                                       | +-------------------------+ |
                                                       | | Docker Container: MySQL | |
                                                       | +-------------------------+ |
                                                       +-----------------------------+
```
**Architecture Highlights** 

Jenkins pulls the pipeline directly from GitHub (Pipeline from SCM)
Flask and MySQL run as separate containers
Containers communicate over a custom Docker network
MySQL uses a persistent Docker volume
Flask starts only after MySQL becomes healthy


**🔄 CI/CD Workflow**
- Code is pushed to the main branch on GitHub
- Jenkins automatically pulls the repository
- Jenkins builds a Docker image for the Flask application

Docker Compose deploys:
- Flask application container
- MySQL database container with persistent storage
- Flask waits for MySQL to pass its health check
- Application becomes available on port 5000
- The pipeline is idempotent, meaning it can be safely re-run without manual cleanup.

**🛠️ Tech Stack**
- CI/CD: Jenkins (Declarative Pipeline)
- Containerization: Docker
- Orchestration: Docker Compose
- Backend: Flask (Python)
- Database: MySQL 5.7
- Source Control: GitHub
- Runtime Environment: Linux (Jenkins host)

**📂 Project Structure**
.
├── Jenkinsfile
├── Dockerfile
├── docker-compose.yml
├── app.py
├── requirement.txt
└── README.md

**Step 1: AWS EC2 Instance Preparation**

Launch EC2 Instance:
- Navigate to the AWS EC2 console.
- Launch a new instance using the Ubuntu 22.04 LTS AMI.
- Select the t2.micro instance type for free-tier eligibility.
- Create and assign a new key pair for SSH access.

<img width="2880" height="1264" alt="image" src="https://github.com/user-attachments/assets/6c16a4d0-bcb2-424b-8acc-a28fcea43d85" />


Configure Security Group:
- Create a security group with the following inbound rules:
- Type: SSH, Protocol: TCP, Port: 22, Source: Your IP
- Type: HTTP, Protocol: TCP, Port: 80, Source: Anywhere (0.0.0.0/0)
- Type: Custom TCP, Protocol: TCP, Port: 5000 (for Flask), Source: Anywhere (0.0.0.0/0)
- Type: Custom TCP, Protocol: TCP, Port: 8080 (for Jenkins), Source: Anywhere (0.0.0.0/0)

<img width="2880" height="1800" alt="image" src="https://github.com/user-attachments/assets/00474b85-095f-45dc-897b-289174551bbf" />

Connect to EC2 Instance:
- Use SSH to connect to the instance's public IP address.
```
ssh -i /path/to/key.pem ubuntu@<ec2-public-ip>
```
**Step 2: Install Dependencies on EC2**

Update System Packages:
```
sudo apt update && sudo apt upgrade -y
Install Git, Docker, and Docker Compose:

sudo apt install git docker.io docker-compose-v2 -y
Start and Enable Docker:

sudo systemctl start docker
sudo systemctl enable docker
Add User to Docker Group (to run docker without sudo):

sudo usermod -aG docker $USER
newgrp docker
```
5. Step 3: Jenkins Installation and Setup

Install Java (OpenJDK 17):
```
sudo apt install openjdk-17-jdk -y
```
Add Jenkins Repository and Install:
```
curl -fsSL [https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key](https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key) | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] [https://pkg.jenkins.io/debian-stable](https://pkg.jenkins.io/debian-stable) binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins -y
```
Start and Enable Jenkins Service:
```
sudo systemctl start jenkins
sudo systemctl enable jenkins
```
Initial Jenkins Setup:

Retrieve the initial admin password:
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Access the Jenkins dashboard at http://<ec2-public-ip>:8080.
- Paste the password, install suggested plugins, and create an admin user. login

<img width="2880" height="1800" alt="image" src="https://github.com/user-attachments/assets/0f6e259e-535d-476b-b114-40ac61f44280" />

- Grant Jenkins Docker Permissions:
```
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

**Step 4: GitHub Repository Configuration**

Ensure your GitHub repository contains the following three files.

Dockerfile

This file defines the environment for the Flask application container.
```
# Use an official Python runtime as a parent image
FROM python:3.9-slim

# Set the working directory in the container
WORKDIR /app

# Install system dependencies required for mysqlclient
RUN apt-get update && apt-get install -y gcc default-libmysqlclient-dev pkg-config && \
    rm -rf /var/lib/apt/lists/*

# Copy the requirements file to leverage Docker cache
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy the rest of the application code
COPY . .

# Expose the port the app runs on
EXPOSE 5000

# Command to run the application
CMD ["python", "app.py"]
```

Docker-compose.yml

This file defines and orchestrates the multi-container application (Flask and MySQL).
```
version: "3.8"

services:
  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: testdb
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - two-tier-nt
    restart: always
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-proot"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s

  flask-app:
    build: .
    ports:
      - "5000:5000"
    depends_on:
      mysql:
        condition: service_healthy
    networks:
      - two-tier-nt
    restart: always

volumes:
  mysql_data:

networks:
  two-tier-nt:
  ```

Jenkinsfile

This file contains the pipeline-as-code definition for Jenkins.
```
pipeline {
    agent any
    stages {
        stage('Clone Code') {
            steps {
                // Replace with your GitHub repository URL
                git branch: 'main', url: '[https://github.com/your-username/your-repo.git](https://github.com/your-username/your-repo.git)'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t flask-app:latest .'
            }
        }
        stage('Deploy with Docker Compose') {
            steps {
                // Stop existing containers if they are running
                sh 'docker compose down || true'
                // Start the application, rebuilding the flask image
                sh 'docker compose up -d --build'
            }
        }
    }
}
```

Step 5: Jenkins Pipeline Creation and Execution

Create a New Pipeline Job in Jenkins:
- From the Jenkins dashboard, select New Item.
- Name the project, choose Pipeline, and click OK.

Configure the Pipeline:
- In the project configuration, scroll to the Pipeline section.
- Set Definition to Pipeline script from SCM.
- Choose Git as the SCM.
- Enter your GitHub repository URL.
- Verify the Script Path is Jenkinsfile.
- Save the configuration.

<img width="1440" height="900" alt="jenkins pipleline config" src="https://github.com/user-attachments/assets/5ba6fae8-453d-4b5a-a6f8-258d52e1ceb0" />

Run the Pipeline:
- Click Build Now to trigger the pipeline manually for the first time.
- Monitor the execution through the Stage View or Console Output.



Verify Deployment:
- After a successful build, your Flask application will be accessible at http://<your-ec2-public-ip>:5000.
- Confirm the containers are running on the EC2 instance with docker ps.

**Conclusion**
The CI/CD pipeline is now fully operational. Any git push to the main branch of the configured GitHub repository will automatically trigger the Jenkins pipeline, which will build the new Docker image and deploy the updated application, ensuring a seamless and automated workflow from development to production.

**🧠 Lessons Learned**
1. CI/CD Pipelines Fail Before Code Runs
Many failures occurred before application code executed, including Git branch mismatches, Jenkins SCM configuration issues, and redundant repository checkouts.
2. Idempotency Is Critical
Hard-coded container names caused repeated pipeline failures. Removing them ensured deployments could be safely re-run without manual cleanup.
3. Tooling Versions Matter
Differences between Docker Compose v1 and v2 caused runtime errors. Aligning pipeline commands with the Jenkins host environment was essential.
4. YAML Is a Production Risk
Invisible formatting issues (tabs vs spaces) completely broke the pipeline, reinforcing the importance of editor configuration.
5. Service Dependencies Must Be Explicit
The application initially failed due to the database not being ready. Health checks and conditional startup logic eliminated race conditions.
6. Logs Are the Source of Truth
All issues were resolved by analyzing Jenkins logs, Docker output, and error messages instead of relying on guesswork.


**🔮 Future Improvements**
- Push Docker images to Docker Hub or AWS ECR
- Replace MySQL container with AWS RDS
- Add automated testing stage to the pipeline
- Migrate CI/CD to GitHub Actions
- Deploy containers to AWS ECS or Kubernetes (EKS)

**👤 Author**
- John Hollingsworth
- Cloud / DevOps Engineer
- Date: 1/2/26 
