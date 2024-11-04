# Mutillidae Project - Capstone Project

## Project Overview

This was initially forked from https://github.com/webpwnized/mutillidae-docker

This project is a DevOps setup for the **Mutillidae** web application, an intentionally vulnerable web application designed for testing security tools and training. This repository configures a CI/CD pipeline to automate code scanning, retrieves secrets from AWS Secret Manager, Docker image building, vulnerability scanning, and deployment to a Kubernetes cluster running on a Minikube instance on an EC2 server with a DAST using OWASP ZAP.

---

## Architecture

The project architecture involves:

1. **Jenkins Pipeline**: Automates the workflow from code checkout to deployment.
2. **SonarCloud**: Used for static code analysis (SAST) to detect potential vulnerabilities in the codebase.
3. **Trivy**: Scans Docker images for vulnerabilities before pushing them to Docker Hub.
4. **Minikube on EC2**: Runs the Kubernetes cluster where the application is deployed and exposed as services.
8. **OWASP ZAP**: Used for Dynamic Application Security Testing for analyzing the deployed web application through the front-end to find vulnerabilities

---
## Requirements

## Software and Accounts
1. **Jenkins**: For CI/CD automation.
2. **Docker**: To build and scan images.
3. **AWS CLI**: For accessing AWS Secrets Manager and EC2.
4. **kubectl**: For managing the Kubernetes deployment on Minikube.
5. **SonarCloud**: Account and token for static analysis.
6. **AWS Secrets Manager**: For securely managing sensitive information (e.g., DockerHub credentials, EC2 credentials, SonarCloud credentials, server IP).

## Prerequisites
1. Access to an AWS account to set up Secrets Manager and EC2.
2. Docker Hub account for storing Docker images.
3. SonarCloud account for code analysis.
   
---
## Installation and Setup
1. **Clone the repository**:
  ```bash
  git clone https://github.com/ToluGIT/ever.git
  cd mutillidae-docker
```
2. **Setup Jenkins Pipeline**:
  Create a new Jenkins pipeline job.
  Copy the Jenkinsfile from this repository and use it for the pipeline configuration.
  Ensure the following are installed and configured on  the Jenkins agent:
    1. AWS CLI
    2. Docker
    3. SonarCloud Scanner 
 and the following necessary plugins installed on Jenkins:
    1. AWS CLI Plugin
    2. GIT plugin
       
3. **Configure AWS Secrets Manager**:
  Sensitive information is managed using AWS Secrets Manager. 
  Store the following secrets in JSON format:

    **DockerHubCredentials**:
     DockerHub credentials for authenticating Docker image pushes.
   
     ```json
      {
        "username": "your-dockerhub-username",
        "password": "your-dockerhub-password"
      }
    ```
  
    **EC2InstanceCredentials**:
      EC2 instance credentials for SSH access to the Minikube server. 
     ```json
      {
        "username": "ec2-user",
        "password": "your-ec2-password"
      }
    ```
    **SonarCloudCredentials**:
      SonarCloud credentials for static analysis.
      
     ```json
        {
          "token": "your-sonar-token",
          "project_key": "your-project-key",
          "organization": "your-organization"
        }
    ```
    **ServerIP**:
      Server IP for remote deployment access.
      
    ```json
        {
          "ip_address": "your-server-ip"
        }
     ```
   Secrets are accessed using the 'aws' CLI in the Jenkins pipeline, ensuring secure and centralized credential management. Configure the assigned role with least privilege. 
    
4. **Set up Environment Variables (on Jenkins)**:
  AWS_REGION: Set to the region of your AWS Secrets Manager (e.g., us-east-1).

--- 

## Pipeline Details
The CI/CD pipeline automates the following stages:

1. Retrieve Secrets:

  - Retrieves sensitive information (credentials, tokens) from AWS Secrets Manager.
    
2. Checkout Code:

  - Clones the repository and checks out the main branch.
    
3. SonarCloud SAST Analysis:

   - Runs static code analysis using SonarCloud to identify code quality issues and security vulnerabilities.

4. Build Docker Images:

   - Builds Docker images using a 'docker-compose.yml' file.
     
5. Trivy Image Scanning:

  - Scans Docker images for high and critical vulnerabilities before pushing them to Docker Hub.
  - The scan artefacts for each docker image are generated and archived for auditing purposes.
    
6. Tag and Push Docker Images to Docker Hub:

  - Tags Docker images with a unique version and commit hash, then pushes them to Docker Hub.
    
7. Deploy to Minikube on EC2:

  - Deploys the Docker containers as Kubernetes deployments on Minikube and exposes them as services.

---

## Usage

To run the pipeline:

1. Trigger the Jenkins pipeline manually or configure it to run on each commit to the main branch.
2. Monitor the pipeline stages in Jenkins to see the build, scan, and deployment progress.
3. Access the deployed application via the 'NodePort' exposed by the Kubernetes service.
---

## Project Services and Port Configuration
Once the containers are up and running, the following services are available on the local network:

  - Port 80, 8080: Mutillidae HTTP web interface
  - Port 81: MySQL Admin HTTP web interface
  - Port 82: LDAP Admin web interface
  - Port 3306: Database interface (MySQL)
  - Port 389: LDAP interface

## Port Forwarding for Local Access
To access services on your local machine, port forwarding has been configured. For example, to access the www (Mutillidae web interface) service, you can forward the port like this:

  ```bash
kubectl port-forward svc/www 8082:80 &
```
In this example, '8082' is the local port on your machine that will map to port '80' on the service. You can replace '8082' with any other available port of your choice if '8082' is already in use.

You can test if the web site is responsive

  ```bash
curl http://127.0.0.1:8888/;
```

## Initial Setup for Mutillidae Database
The first time you access the Mutillidae web interface, you may see a warning page indicating that the database cannot be found. This is expected behavior. Use the provided link on that page to "rebuild" the database, and it should begin working normally.

Alternatively, you can automate this initial database setup with the following command:

  ```bash
curl http://127.0.0.1:8082/set-up-database.php
```
This command triggers the creation of the required database for Mutillidae.

---
## Security Testing with OWASP ZAP
For security testing, an OWASP ZAP (Zed Attack Proxy) DAST (Dynamic Application Security Testing) scan is run directly on the EC2 instance where Minikube is set up. This scan assesses the security of the deployed web application by simulating attacks on the live environment.

## Running the OWASP ZAP DAST Scan
Before running the OWASP ZAP scan, ensure that the www service is accessible from the local Docker environment on the Minikube instance. 

Once port forwarding is set up, the OWASP ZAP scan can be initiated from the Minikube instance using the following Docker command:
  ```bash
docker run -v ${PWD}:/zap/wrk/:rw zaproxy/zap-stable zap-baseline.py -t http://host.docker.internal:8082 -r scan-report.html -d
```
This command:

- Mounts the current directory ('${PWD}') on the Minikube instance to '/zap/wrk/' in the OWASP ZAP container, where the scan report will be saved.
- Runs the OWASP ZAP 'zap-baseline.py' script, targeting the deployed application at 'http://host.docker.internal:8082' (adjust the port if you used a different one).
- Generates a report named 'scan-report.html', which will be saved in the mounted directory.

## Viewing the Report
The scan report, scan-report.html, will be saved in the directory you specified when mounting (${PWD} in the example). You can open this file in a web browser to review the vulnerabilities and issues detected during the scan.

This section guides users on setting up the OWASP ZAP DAST scan on the EC2 instance and provides instructions on generating and accessing the security scan report.

--- 

## Troubleshooting
1. Docker login issues: If Docker credentials are not retrieved correctly, ensure that the DockerHubCredentials secret in AWS Secrets Manager is formatted correctly and that Jenkins has the correct permissions.

2. AWS Secrets Manager errors: If secrets fail to load, check AWS IAM permissions for the Jenkins instance and ensure that secrets are stored in the correct format.

3. Kubernetes deployment issues: If Minikube deployments fail, ensure that Minikube is running on the EC2 instance.

4. SonarCloud analysis failures: Verify that the SonarCloud credentials in AWS Secrets Manager are valid and that SonarCloud Scanner was installed on jenkins correctly.

5. Permission errors: Ensure that sshpass and aws CLI are properly installed on the Jenkins server and that Jenkins has sufficient permissions to execute them.

--- 

## Conclusion
This project deploys the Mutillidae application onto a Minikube instance in an EC2 environment, integrating security checks into a Jenkins pipeline. By using tools like SonarCloud for static analysis and OWASP ZAP for dynamic security testing, it demonstrates secure DevOps practices (DevSecOps).

Next Steps
To enhance the project:

1. Expand Security: Add more security scanners and set up automated scans.
2. Improve Monitoring: Integrate logging/monitoring tools like ELK or Prometheus.
3. Optimize Performance: Conduct load testing and streamline the CI/CD process.
4. Extend Deployment: Support multiple environments (e.g., AWS EKS, GKE).

## Acknowledgments
This project follows DevOps security best practices. Thanks to the open-source community for the tools that made this possible.

Thank you for exploring this project! We hope it serves as a helpful example of secure DevOps integration.
