<a id="top"></a>
# 🚀 AWS EKS + Jenkins CI/CD — Project Workflow Execution.

> **Repository:** [myweb](https://github.com/sm-simplifies/myweb.git)

---

## 🧩 About the Project
This project demonstrates a **DevOps pipeline** integrating **Jenkins**, **Docker**, and **AWS EKS (Kubernetes)** to achieve **Continuous Integration and Continuous Deployment (CI/CD)** for a Java web application. The application is containerized using **Tomcat**, monitored with **Prometheus** and **Grafana**, and automatically deployed to **EKS** using a Jenkins pipeline.

---

## 📚 Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Prepare AWS and IAM](#prepare-aws-and-iam)
4. [Launch EC2 for Jenkins](#launch-ec2-for-jenkins)
5. [Install Required Software](#install-required-software)
6. [Configure Jenkins](#configure-jenkins)
7. [Create IAM Roles for EKS](#create-iam-roles-for-eks)
8. [Create EKS Cluster](#create-eks-cluster)
9. [Jenkins Pipeline Explanation](#jenkins-pipeline-explanation)
10. [Run & Verify Deployment](#verify-deployment)
11. [Troubleshooting](#troubleshooting)
12. [Appendix: Files](#appendix-files)

---

<a id="overview"></a>
## 🧭 Overview
- 🔁 **Jenkins** automates: Code → Build → Dockerize → Push → Deploy.
- 🐳 **Docker Hub** hosts the built image (`smicx20/myweb-image`).
- ☸️ **AWS EKS** runs the application in Kubernetes pods.
- 📊 **Prometheus** and **Grafana** provide monitoring and visualization.

---

<a id="prerequisites"></a>
## ⚙️ Prerequisites
- ✅ AWS Account with required permissions (EC2, EKS, IAM).
- ✅ Docker Hub account: `mayrhatte09`.
- ✅ GitHub repository: [myweb](https://github.com/sm-simplifies/myweb.git).
- ✅ Local setup or EC2 instance with AWS CLI and kubectl installed.

---

<a id="prepare-aws-and-iam"></a>
## 🧱 1. Prepare AWS and IAM
1. Go to **AWS Console → IAM**.
2. Create an **IAM User** with *programmatic access*.
3. Save **Access Key ID** and **Secret Key**.
4. Attach permissions for EC2, EKS, and S3.
5. These credentials will later be used for Jenkins configuration.

---

<a id="launch-ec2-for-jenkins"></a>
## ☁️ 2. Launch EC2 for Jenkins
| Parameter | Value |
|------------|-------|
| **AMI** | Amazon Linux 2 |
| **Instance Type** | t3.large |
| **Storage** | 30 GiB |
| **Ports** | 22 (SSH), 8080 (Jenkins), 80/443 (optional) |

🔑 Create or select an SSH key pair for EC2 access.

---

<a id="install-required-software"></a>
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

<a id="configure-jenkins"></a>
## 🧩 4. Configure Jenkins
1. Unlock Jenkins using `/var/lib/jenkins/secrets/initialAdminPassword`.
2. Install **recommended plugins** (Git, Pipeline, Docker Pipeline, Kubernetes).
3. Add credentials:
   - 🐳 **Docker Hub:** Username & password → ID: `dockerhub-pass`

---

<a id="create-iam-roles-for-eks"></a>
## 🔐 5. Create IAM Roles for EKS
1. **Master Role:** Use case → EKS Cluster.
2. **Worker Node Role:** Use case → EC2.
   - Attach policies:
     - `AmazonEKS_CNI_Policy`
     - `AmazonEC2ContainerRegistryReadOnly`
     - `AmazonEKSWorkerNodePolicy`

---

<a id="create-eks-cluster"></a>
## ☸️ 6. Create EKS Cluster
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
aws eks update-kubeconfig --region ap-southeast-1 --name master-node
kubectl get nodes
```

---

<a id="jenkins-pipeline-explanation"></a>
## 🧩 7. Jenkins Pipeline Explanation

The provided Jenkinsfile stages: 
- Git Checkout: clone the repo. 
- Maven Build: mvn clean package to produce the WAR. 
- Docker Build: build image myweb-image:v${BUILD_NUMBER} . 
- Docker Login & Push: login to Docker Hub (credential dockerhub-pass), tag and push. 
- Update Deployment File: update deployments.yaml to new tag using sed. 
- Kubernetes Deployment: apply deployments.yaml 
Important Jenkins credential IDs used in the Jenkinsfile must match those created earlier.

---

<a id="verify-deployment"></a>
## ✅ 8. Run & Verify Deployment
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
   🌍 Open: [http://<PublicIP>:8080](http://<PublicIP>:8080)

---

<a id="troubleshooting"></a>
## 🧠 9. Troubleshooting
| Problem | Fix |
|----------|------|
| **ImagePullBackOff** | Ensure image tag matches Docker Hub tag. |
| **kubectl not found** | Confirm `/usr/bin/kubectl` exists & executable. |
| **Permission denied (Docker)** | Restart Jenkins after `usermod -aG docker jenkins`. |
| **AWS CLI error** | Re-run `aws configure` with correct keys. |
| **Pods stuck pending** | Verify node role and subnet permissions. |

---

<a id="appendix-files"></a>
## 📁 Appendix: Files

### [deployments.yaml](https://github.com/sm-simplifies/myweb/blob/769f01e33e9ccb480add79c2e53623c73c4f3c67/deployments.yaml)

### [dockerfile](https://github.com/sm-simplifies/myweb/blob/d599e1ec9bb6a1248017a6e0b379b98a8b690fcc/dockerfile)

### [jenkinsfile](https://github.com/sm-simplifies/myweb/blob/abfff0a0b5145fdd37d6211759c0496b228d84cc/jenkinsfile)

---

## 👨‍💻 Author
**Swapnil Mali** — AWS & DevOps Engineer  
💡 *"Knowledge should spread!"* 💪

---

[TOP](#top)


