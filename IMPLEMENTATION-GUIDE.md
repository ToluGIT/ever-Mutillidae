# Mutillidae DevSecOps Implementation Guide

This guide provides step-by-step instructions to implement the improved Mutillidae DevSecOps pipeline with security fixes and best practices.

## Issues Fixed

### Critical Issues Resolved:
1. **Database Port Mismatch** - Fixed K8s configs to use MySQL port 3306 instead of PostgreSQL 5432
2. **Inconsistent Deployments** - Replaced imperative `kubectl create` with declarative YAML files
3. **Security Vulnerabilities** - Removed hard-coded credentials and implemented SSH key authentication
4. **Missing Health Checks** - Added deployment validation and health monitoring
5. **Template Placeholders** - Fixed `{{IMAGE_NAME}}` placeholders in K8s YAML files

## Prerequisites

### Software Requirements
- Jenkins with required plugins (AWS CLI, Git, Docker)
- AWS CLI configured on Jenkins agent
- Docker installed on Jenkins agent
- SonarCloud Scanner installed on Jenkins agent
- SSH client tools (scp, ssh)

### AWS Setup Required
1. **AWS Secrets Manager** with updated secrets format
2. **EC2 Instance** running Minikube
3. **IAM Role** with appropriate permissions

## Implementation Steps

### Step 1: Update AWS Secrets Manager

Update your AWS Secrets Manager secrets with the new format:

#### EC2InstanceCredentials (NEW FORMAT - SSH Key Based)
```json
{
  "username": "ec2-user",
  "ssh_private_key": "-----BEGIN RSA PRIVATE KEY-----\nYOUR_PRIVATE_KEY_CONTENT_HERE\n-----END RSA PRIVATE KEY-----"
}
```

> **Important**: Remove the old password-based authentication and use SSH keys for security.

#### Other Secrets (Keep Same Format)
- **DockerHubCredentials**: `{"username": "your-username", "password": "your-password"}`
- **SonarCloudCredentials**: `{"token": "your-token", "project_key": "your-key", "organization": "your-org"}`
- **ServerIP**: `{"ip_address": "your-server-ip"}`

### Step 2: Prepare EC2 Instance

On your EC2 instance running Minikube:

1. **Add SSH Public Key**:
   ```bash
   # Add your public key to ~/.ssh/authorized_keys
   echo "your-public-key-content" >> ~/.ssh/authorized_keys
   chmod 600 ~/.ssh/authorized_keys
   ```

2. **Verify Minikube is Running**:
   ```bash
   minikube status
   kubectl cluster-info
   ```

3. **Ensure kubectl Access**:
   ```bash
   kubectl get nodes
   ```

### Step 3: Deploy Updated Pipeline

1. **Update Repository**: 
   ```bash
   git add .
   git commit -m "Fix critical DevSecOps pipeline issues"
   git push origin main
   ```

2. **Update Jenkins Pipeline**:
   - Copy the updated `jenkinsfile` content
   - Update your Jenkins pipeline configuration
   - Ensure all required plugins are installed

### Step 4: Test the Pipeline

1. **Trigger Pipeline**:
   - Run the Jenkins pipeline manually
   - Monitor all stages for successful completion

2. **Verify Deployment**:
   ```bash
   # On EC2 instance, check deployment status
   kubectl get deployments
   kubectl get pods
   kubectl get services
   ```

3. **Test Application Access**:
   ```bash
   # Port forward to test locally
   kubectl port-forward svc/www-service 8080:80 &
   curl http://localhost:8080
   ```

## Pipeline Stages Explained

### 1. **Retrieve Secrets from AWS Secrets Manager**
- Fetches all credentials securely from AWS Secrets Manager
- Now uses SSH private key instead of passwords

### 2. **Checkout Code**
- Clones repository and gets commit hash for image tagging

### 3. **SonarCloud SAST Analysis**
- Performs static code analysis for security vulnerabilities
- Blocks deployment if critical issues found

### 4. **Build Docker Images**
- Builds all service images using docker-compose
- Images are tagged with build number and commit hash

### 5. **Trivy Image Scanning**
- Scans images for high/critical vulnerabilities
- Generates scan reports for auditing

### 6. **Tag and Push Docker Images to Docker Hub**
- Tags images with version information
- Securely pushes to Docker registry

### 7. **Deploy to Minikube on EC2**
- **NEW**: Uses declarative K8s YAML files
- **NEW**: Updates image placeholders with actual tags
- **NEW**: Uses SSH keys for secure authentication
- **NEW**: Applies secrets and configmaps first

### 8. **Health Check and Validation** *(NEW STAGE)*
- Waits for all deployments to be ready
- Validates pod and service status
- Tests internal connectivity

##Security Improvements

### Authentication
- SSH key-based authentication (replaces passwords)
-  Kubernetes secrets for sensitive data
- AWS Secrets Manager integration

### Network Security
- StrictHostKeyChecking with proper key management
- ClusterIP services for internal communication
- NodePort only for external-facing services

### Pipeline Security
- No secrets in logs or command line
- Temporary files cleaned up after use
- Proper error handling and troubleshooting

## Monitoring and Troubleshooting

### Health Check Commands
```bash
# Check deployment status
kubectl get deployments -o wide

# Check pod status
kubectl get pods -o wide

# View service endpoints
kubectl get services

# Check pod logs
kubectl logs -l app=www --tail=50

# Describe problematic pods
kubectl describe pods -l app=www
```

### Common Issues and Solutions

#### 1. **Deployment Timeout**
```bash
# Check pod events
kubectl describe pod <pod-name>

# Check resource limits
kubectl top pods
```

#### 2. **Image Pull Errors**
```bash
# Verify image exists in registry
docker pull docker.io/toluid/www:v1-abc123

# Check imagePullSecrets if needed
kubectl describe pod <pod-name>
```

#### 3. **Service Connectivity Issues**
```bash
# Test service resolution
kubectl run test-pod --image=busybox --rm -it --restart=Never -- nslookup database-service

# Check service endpoints
kubectl get endpoints
```

## Maintenance and Updates

### Regular Tasks
1. **Update base images** in Dockerfiles for security patches
2. **Rotate SSH keys** periodically
3. **Review and update** Trivy scan results
4. **Monitor** SonarCloud quality gates
5. **Backup** persistent volumes if using them

### Scaling and Improvements
1. **Implement Helm charts** for better template management
2. **Resource limits added** - CPU/memory limits implemented
3. **Set up monitoring** with Prometheus/Grafana
4. **Implement blue-green deployment** strategy
5. **Add automated rollback** on health check failures

## Docker Security Fixes Applied

### **Hardcoded Password Removal**
Removed hardcoded passwords from all Dockerfiles:
- **database/Dockerfile**: Removed `ENV MYSQL_ROOT_PASSWORD="mutillidae"`
- **database_admin/Dockerfile**: Removed `ENV PMA_PASSWORD="mutillidae"`  
- **ldap/Dockerfile**: Removed `ENV LDAP_ADMIN_PASSWORD="mutillidae"`
- **www/Dockerfile**: Removed default value from `ARG DATABASE_PASSWORD`

### **Runtime Environment Variables**
All containers now receive passwords via:
- **Kubernetes Secrets** - Secure password injection at runtime
- **ConfigMaps** - Non-sensitive configuration values
- **Build Arguments** - For www container database config

### **Resource Management**
Added CPU and memory limits to all deployments:
```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "200m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

## Next Steps
After successful implementation:

1. **Set up monitoring** (Prometheus/Grafana)
2. **Implement logging** (ELK stack)
3. **Add integration tests** post-deployment
4. **Set up alerts** for deployment failures
5. **Implement automated rollbacks**
6. **Consider GitOps** with ArgoCD or Flux

## Important Notes

- **Test in staging first** before production deployment
- **Backup your data** before making changes
- **Monitor resource usage** on EC2 instance
- **Keep SSH keys secure** and rotate regularly
- **Review security scan results** before deployment

