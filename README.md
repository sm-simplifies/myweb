# 🚀 AWS EKS + Jenkins CI/CD Pipeline

> **Repository:** [myweb](https://github.com/sm-simplifies/myweb.git)

---

## 🧩 About the Project
This project demonstrates a **DevOps pipeline** integrating **Jenkins**, **Docker**, and **AWS EKS (Kubernetes)** to achieve **Continuous Integration and Continuous Deployment (CI/CD)** for a Java web application. The application is containerized using **Tomcat**, monitored with **Prometheus** and **Grafana**, and automatically deployed to **EKS** using a Jenkins pipeline.

---

## 📚 Table of Contents
1. [Overview](##overview)
2. [Prerequisites](#prerequisites)
3. [Prepare AWS & IAM](#1-prepare-aws--iam)
4. [Launch EC2 for Jenkins](#2-launch-ec2-for-jenkins)
5. [Install Required Software](#3-install-required-software)
6. [Configure Jenkins](#4-configure-jenkins)
7. [Build Docker Image](#5-build-docker-image)
8. [Create IAM Roles for EKS](#6-create-iam-roles-for-eks)
9. [Create EKS Cluster](#7-create-eks-cluster)
10. [Deploy to Kubernetes](#8-deploy-to-kubernetes)
11. [Pipeline Explanation](#9-jenkins-pipeline-explanation)
12. [Verify Deployment](#10-verify-deployment)
13. [Troubleshooting](#11-troubleshooting)
14. [Appendix: Files](#appendix-files)

---

## 🧭 Overview
- 🔁 **Jenkins** automates: Code → Build → Dockerize → Push → Deploy.
- 🐳 **Docker Hub** hosts the built image (`mayrhatte09/myimage`).
- ☸️ **AWS EKS** runs the application in Kubernetes pods.
- 📊 **Prometheus** and **Grafana** provide monitoring and visualization.

---

## ⚙️ Prerequisites
- ✅ AWS Account with required permissions (EC2, EKS, IAM).
- ✅ Docker Hub account: `mayrhatte09`.
- ✅ GitHub repository: [myweb](https://github.com/Mayurhatte09/myweb.git).
- ✅ Local setup or EC2 instance with AWS CLI and kubectl installed.

---

## 🧱 1. Prepare AWS & IAM
1. Go to **AWS Console → IAM**.
2. Create an **IAM User** with *programmatic access*.
3. Save **Access Key ID** and **Secret Key**.
4. Attach permissions for EC2, EKS, and S3.
5. These credentials will later be used for Jenkins configuration.

---

## ☁️ 2. Launch EC2 for Jenkins
| Parameter | Value |
|------------|-------|
| **AMI** | Amazon Linux 2 |
| **Instance Type** | t3.large |
| **Storage** | 30 GiB |
| **Ports** | 22 (SSH), 8080 (Jenkins), 80/443 (optional) |

🔑 Create or select an SSH key pair for EC2 access.

---

## 🧰 3. Install Required Software
SSH into EC2 and run:

```bash
sudo yum update -y
sudo yum install -y java-11-amazon-corretto docker maven

# Jenkins setup
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
sudo yum install -y jenkins
sudo systemctl enable --now jenkins

# Docker permissions
sudo systemctl enable --now docker
sudo usermod -aG docker jenkins

# kubectl installation
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo chmod +x kubectl
sudo mv kubectl /usr/bin/

sudo usermod -s /bin/bash jenkins
grep jenkins /etc/passwd
```

🌐 Access Jenkins: `http://<EC2_PUBLIC_IP>:8080`

---

## 🧩 4. Configure Jenkins
1. Unlock Jenkins using `/var/lib/jenkins/secrets/initialAdminPassword`.
2. Install **recommended plugins** (Git, Pipeline, Docker Pipeline, Kubernetes).
3. Add credentials:
   - 🐳 **Docker Hub:** Username & password → ID: `dockerhub-pass`
   - ☁️ **AWS (Optional):** IAM Access Key / Secret Key
4. Test Jenkins → New Job → `docker ps`

---

## 🐋 5. Build Docker Image
**Dockerfile:**
```dockerfile
FROM tomcat:9.0.109
COPY target/myweb*.war /usr/local/tomcat/webapps/myweb.war
```

📦 The pipeline compiles WAR → builds Docker image → pushes to Docker Hub.

---

## 🔐 6. Create IAM Roles for EKS
1. **Master Role:** Use case → EKS Cluster.
2. **Worker Node Role:** Use case → EC2.
   - Attach policies:
     - `AmazonEKS_CNI_Policy`
     - `AmazonEC2ContainerRegistryReadOnly`
     - `AmazonEKSWorkerNodePolicy`

---

## ☸️ 7. Create EKS Cluster
```bash
eksctl create cluster \
  --name moster-node \
  --version 1.27 \
  --region ap-southeast-1 \
  --nodegroup-name worker-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 3
```

Then configure:
```bash
aws eks update-kubeconfig --region ap-southeast-1 --name moster-node
kubectl get nodes
```

---

## 📦 8. Deploy to Kubernetes
Apply the manifest:
```bash
kubectl apply -f deployments.yaml
kubectl get pods -o wide
```

---

## 🧩 9. Jenkins Pipeline Explanation
**Jenkinsfile:**
```groovy
pipeline {
  agent any
  tools { maven "Apache Maven 3.8.4" }
  environment {
    DOCKER_HUB_USER = 'mayrhatte09'
    IMAGE_NAME = 'myimage'
  }
  stages {
    stage('Git Checkout') { steps { git url: 'https://github.com/Mayurhatte09/myweb.git', branch: 'main' } }
    stage('Maven Build') { steps { sh 'mvn clean package' } }
    stage('Docker Build') { steps { sh 'docker build -t ${IMAGE_NAME}:v${BUILD_NUMBER} .' } }
    stage('Docker Push') {
      steps {
        withCredentials([string(credentialsId: 'dockerhub-pass', variable: 'DOCKER_HUB_PASS')]) {
          sh '''
            echo "$DOCKER_HUB_PASS" | docker login -u "$DOCKER_HUB_USER" --password-stdin
            docker tag ${IMAGE_NAME}:v${BUILD_NUMBER} ${DOCKER_HUB_USER}/${IMAGE_NAME}:v${BUILD_NUMBER}
            docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:v${BUILD_NUMBER}
          '''
        }
      }
    }
    stage('Deploy to K8s') {
      steps {
        sh '''
          aws eks update-kubeconfig --region ap-southeast-1 --name moster-node
          sed -i "s|${DOCKER_HUB_USER}/${IMAGE_NAME}:v[0-9]*|${DOCKER_HUB_USER}/${IMAGE_NAME}:v${BUILD_NUMBER}|g" deployments.yaml
          kubectl apply -f deployments.yaml
          kubectl rollout restart deployment mywebdeployment
          kubectl get pods -o wide
        '''
      }
    }
  }
}
```

---

## ✅ 10. Verify Deployment
1. Run Jenkins pipeline.
2. Check Docker Hub for image tag `v{BUILD_NUMBER}`.
3. Validate deployment:
   ```bash
   kubectl get pods -o wide
   kubectl get svc
   ```
4. Access app:
   ```bash
   kubectl port-forward svc/myweb-service 8080:8080
   ```
   🌍 Open: [http://localhost:8080](http://localhost:8080)

---

## 🧠 11. Troubleshooting
| Problem | Fix |
|----------|------|
| **ImagePullBackOff** | Ensure image tag matches Docker Hub tag. |
| **kubectl not found** | Confirm `/usr/bin/kubectl` exists & executable. |
| **Permission denied (Docker)** | Restart Jenkins after `usermod -aG docker jenkins`. |
| **AWS CLI error** | Re-run `aws configure` with correct keys. |
| **Pods stuck pending** | Verify node role and subnet permissions. |

---

## 📁 Appendix: Files
### deployments.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mywebdeployment
  labels:
    app: myweb
spec:
  replicas: 4
  selector:
    matchLabels:
      app: myweb
  template:
    metadata:
      labels:
        app: myweb
    spec:
      containers:
      - name: myweb
        image: mayrhatte09/myimage:v1
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: myweb-service
spec:
  selector:
    app: myweb
  ports:
    - port: 8080
      targetPort: 8080
  type: NodePort
```

### Dockerfile
```dockerfile
FROM tomcat:9.0.109
COPY target/myweb*.war /usr/local/tomcat/webapps/myweb.war
```

---

## 👨‍💻 Author
**Swapnil Mali** — AWS & DevOps Engineer  
💡 *"Knowledge should spread!"* 💪

