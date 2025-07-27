# Mutillidae DevSecOps Project - Step-by-Step Implementation Guide

## Prerequisites

### Required
- Jenkins server with admin access (with Java 17+ for SonarCloud)
- AWS CLI installed on Jenkins agent
- Docker installed on Jenkins agent
- SonarCloud Scanner installed on Jenkins agent (requires Java 17+)
- Trivy scanner installed on Jenkins agent
- EC2 instance running with Minikube installed
- AWS account with Secrets Manager access

### Required Accounts
- AWS account
- Docker Hub account
- SonarCloud account
- GitHub account (or similar Git repository)

## Step 1: Set Up AWS Infrastructure

### 1.1 Create EC2 Instance
```bash
# Launch Ubuntu EC2 instance (t3.medium recommended)
# Ensure security group allows:
# - SSH (port 22) from Jenkins server IP
# - HTTP (port 80) for application access
# - Custom ports 30000-32767 for NodePort services
```

### 1.2 Install Minikube on EC2
```bash
# Connect to EC2 instance
ssh -i your-key.pem ubuntu@your-ec2-ip

# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
sudo apt install -y docker.io
sudo usermod -aG docker $USER
sudo systemctl enable docker
sudo systemctl start docker

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install Minikube
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube /usr/local/bin/

# Start Minikube
minikube start --driver=docker

# Verify installation
kubectl cluster-info
kubectl get nodes
```

## Step 2: Configure AWS Secrets Manager

### 2.1 Create SSH Key Pair
```bash
# Generate SSH key pair on your local machine
ssh-keygen -t rsa -b 4096 -f ~/.ssh/mutillidae-key

# Copy public key to EC2 instance
ssh-copy-id -i ~/.ssh/mutillidae-key.pub ubuntu@your-ec2-ip
```

### 2.2 Store Secrets in AWS Secrets Manager
Create the following secrets in AWS Secrets Manager:

**EC2InstanceCredentials**
```json
{
  "username": "ubuntu"
}
```

**Note**: SSH private key should be stored separately in Jenkins credentials manager as 'ec2-ssh-key' for security reasons.

**DockerHubCredentials**
```json
{
  "username": "your-dockerhub-username",
  "password": "your-dockerhub-password"
}
```

**SonarCloudCredentials**
```json
{
  "token": "your-sonarcloud-token",
  "project_key": "your-project-key",
  "organization": "your-organization"
}
```

**ServerIP**
```json
{
  "ip_address": "your-ec2-instance-ip"
}
```

## Step 3: Set Up Jenkins

### 3.1 Install Required Jenkins Plugins
- AWS CLI Plugin
- Git plugin
- Docker Pipeline plugin
- Pipeline plugin
- Pipeline Utility Steps plugin (for readJSON)
- SSH Agent plugin (for SSH key credentials)

### 3.2 Configure Jenkins Credentials
- Create AWS IAM role for Jenkins with Secrets Manager read permissions
- Configure AWS credentials in Jenkins
- Add SSH private key as Jenkins credential with ID 'ec2-ssh-key'
- Set AWS_REGION environment variable in Jenkins (e.g., us-east-1)

### 3.3 Install Required Tools on Jenkins Agent
```bash
# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Install Docker
sudo apt update
sudo apt install -y docker.io
sudo usermod -aG docker jenkins

# Install SonarCloud Scanner (requires Java 17+)
# First ensure Java 17 is installed
sudo apt update
sudo apt install -y openjdk-17-jdk

# Download and install SonarCloud Scanner
wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.8.0.2856-linux.zip
unzip sonar-scanner-cli-4.8.0.2856-linux.zip
sudo mv sonar-scanner-4.8.0.2856-linux /usr/local/sonar-scanner
sudo ln -s /usr/local/sonar-scanner/bin/sonar-scanner /usr/local/bin/sonar-scanner

# Install Trivy
sudo apt-get update
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

## Step 4: Prepare the Repository

### 4.1 Clone the Repository
```bash
git clone https://github.com/toluGIT/ever-Mutillidae.git
cd ever-Mutillidae
```

### 4.2 Verify File Structure
Ensure your repository has:
```
├── .build/
│   ├── docker-compose.yml
│   ├── database/Dockerfile
│   ├── database_admin/Dockerfile
│   ├── ldap/Dockerfile
│   ├── ldap_admin/Dockerfile
│   └── www/Dockerfile
├── k8s/
│   ├── secrets.yaml
│   ├── database-deployment.yaml
│   ├── database-service.yaml
│   ├── www-deployment.yaml
│   ├── www-service.yaml
│   ├── ldap-deployment.yaml
│   ├── ldap-service.yaml
│   ├── database_admin-deployment.yaml
│   ├── database_admin-service.yaml
│   ├── ldap_admin-deployment.yaml
│   └── ldap_admin-service.yaml
├── jenkinsfile
└── README.md
```

## Step 5: Configure SonarCloud

### 5.1 Create SonarCloud Project
1. Go to https://sonarcloud.io
2. Login with GitHub/GitLab/Azure DevOps
3. Create new project
4. Get project key and organization key
5. Generate authentication token

### 5.2 Update Repository for SonarCloud
Create `sonar-project.properties` in repository root:
```properties
sonar.projectKey=your-project-key
sonar.organization=your-organization
sonar.host.url=https://sonarcloud.io
sonar.sources=.
sonar.exclusions=**/*.jpg,**/*.png,**/*.gif,**/*.pdf
```

## Step 6: Set Up Docker Hub Repository

### 6.1 Create Docker Hub Repositories
Create repositories for each service:
- your-username/database
- your-username/database_admin
- your-username/www
- your-username/ldap
- your-username/ldap_admin

### 6.2 Update Image Names in Jenkinsfile
Update the `IMAGE_PREFIX` in jenkinsfile:
```groovy
IMAGE_PREFIX = 'your-dockerhub-username'
```

## Step 7: Deploy Kubernetes Secrets

### 7.1 Connect to EC2 Instance
```bash
ssh -i ~/.ssh/mutillidae-key ubuntu@your-ec2-ip
```

### 7.2 Apply Kubernetes Secrets
```bash
# Create namespace (optional)
kubectl create namespace mutillidae

# Apply secrets and configmaps
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: mutillidae-secrets
  namespace: default
type: Opaque
data:
  mysql-root-password: bXV0aWxsaWRhZQ==  # base64 encoded 'mutillidae'
  ldap-admin-password: bXV0aWxsaWRhZQ==   # base64 encoded 'mutillidae'
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mutillidae-config
  namespace: default
data:
  MYSQL_DATABASE: "mutillidae"
  MYSQL_USER: "root"
  LDAP_ORGANISATION: "Mutillidae Inc"
  LDAP_DOMAIN: "mutillidae.localhost"
  LDAP_BASE_DN: "dc=mutillidae,dc=localhost"
EOF
```

## Step 8: Configure Jenkins Pipeline

### 8.1 Create Jenkins Pipeline Job
1. Go to Jenkins dashboard
2. Click "New Item"
3. Enter project name
4. Select "Pipeline"
5. Click "OK"

### 8.2 Configure Pipeline
1. In "Pipeline" section, select "Pipeline script from SCM"
2. Select "Git" as SCM
3. Enter repository URL
4. Set branch to "main"
5. Set script path to "jenkinsfile"
6. Save configuration

## Step 9: Test the Pipeline

### 9.1 Run Initial Build
1. Click "Build Now" in Jenkins
2. Monitor build progress
3. Check console output for errors

### 9.2 Verify Each Stage
Monitor these stages:
1. Retrieve Secrets from AWS Secrets Manager
2. Checkout Code
3. SonarCloud SAST Analysis
4. Build Docker Images
5. Trivy Image Scanning
6. Tag and Push Docker Images to Docker Hub
7. Deploy to Minikube on EC2
8. Health Check and Validation

## Step 10: Verify Deployment

### 10.1 Check Kubernetes Resources
```bash
# Connect to EC2 instance
ssh -i ~/.ssh/mutillidae-key ubuntu@your-ec2-ip

# Check deployments
kubectl get deployments

# Check pods
kubectl get pods -o wide

# Check services
kubectl get services
```

### 10.2 Test Application Access
```bash
# Get service URLs
kubectl get services

# Port forward to test locally (use service name 'www' after DNS fix)
kubectl port-forward svc/www 8080:80 &

# Test application (from another terminal)
curl http://localhost:8080

# Or test from external machine using NodePort
# curl http://your-ec2-ip:nodeport
```

## Step 11: Initialize Database

### 11.1 Access Application
1. Get the www service NodePort: `kubectl get svc www`
2. Access: `http://your-ec2-ip:nodeport`
3. Click link to "rebuild database" or run:
```bash
kubectl port-forward --address 0.0.0.0 service/www 8080:80 &
curl http://localhost:8080/set-up-database.php
```

## Step 12: Run Security Testing

### 12.1 Set Up OWASP ZAP
```bash
# On EC2 instance, ensure port forwarding is active
kubectl port-forward --address 0.0.0.0 service/www 8080:80 &

# Run OWASP ZAP scan (use actual EC2 IP instead of host.docker.internal)
docker run -v $(pwd):/zap/wrk/:rw zaproxy/zap-stable zap-baseline.py \
  -t http://YOUR_EC2_IP:8080 \
  -r scan-report.html \
  -d

# Replace YOUR_EC2_IP with your actual EC2 public IP address
```

### 12.2 Review Security Reports
- Trivy scan reports in Jenkins artifacts
- SonarCloud analysis results
- OWASP ZAP scan report (scan-report.html)

## Step 13: Monitor and Maintain

### 13.1 Regular Monitoring
```bash
# Check pod health
kubectl get pods --watch

# Check resource usage
kubectl top pods
kubectl top nodes

# Check logs
kubectl logs -l app=www --tail=50
```

### 13.2 Troubleshooting Commands
```bash
# Describe problematic pods
kubectl describe pod <pod-name>

# Get detailed events
kubectl get events --sort-by=.metadata.creationTimestamp

# Test internal connectivity
kubectl run test-pod --image=busybox --rm -it --restart=Never -- nslookup database
```

## Step 14: Production Considerations

### 14.1 Security Enhancements
- Rotate SSH keys regularly
- Use least privilege IAM policies
- Enable logging and monitoring
- Implement network policies
- Use private Docker registries

### 14.2 Scalability Improvements
- Implement horizontal pod autoscaling
- Use persistent volumes for data
- Set up ingress controller
- Implement service mesh
- Add monitoring with Prometheus/Grafana

### 14.3 Backup and Recovery
- Backup persistent volumes
- Document recovery procedures
- Test disaster recovery plans
- Implement automated backups


This guide provides a complete walkthrough for implementing the Mutillidae DevSecOps pipeline from scratch. Follow each step carefully and verify completion before proceeding to the next step.