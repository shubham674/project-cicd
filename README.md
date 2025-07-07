# Simple_NodeJS_App
In this video, we'll walk you through building a Complete CI/CD Pipeline using GitHub, Jenkins (running in a container), SonarQube (containerized), NPM, Docker, Trivy, Amazon ECR, and deploying to Amazon ECS with Fargate. This end-to-end pipeline will automate the process from code commit to production deployment with security and quality checks.


## Video Link
Watch the comprehensive tutorial on YouTube: [CI/CD Pipeline with Jenkins, Docker, and AWS ECS](https://www.youtube.com/watch?v=uGWF3meicAk&t=1827s)

---

## Video Summary
**Comprehensive Guide to CI/CD Pipeline with Jenkins, Docker, and AWS ECS**

This tutorial provides a detailed walkthrough of building a CI/CD pipeline using Jenkins, Docker, and AWS ECS, aimed at improving software development efficiency and security.

### **Main Insights**
1. **Setting Up Jenkins**: Automated code builds from GitHub repositories.
2. **SonarQube Integration**: Vulnerability checks to ensure secure applications.
3. **Docker for Image Creation**: Building and managing Docker images.
4. **AWS ECS Deployment**: Using AWS Fargate for simplified infrastructure management.
5. **Resource Management**: Cleaning up resources to prevent unnecessary costs.

---

## Overview of the CI/CD Pipeline
The video demonstrates the creation of an end-to-end Continuous Integration and Continuous Deployment (CI/CD) pipeline with the following steps:

### **Pipeline Steps**
1. **Code Build and Vulnerability Checks**
   - Jenkins setup for GitHub integration.
   - Running unit tests and scanning for vulnerabilities using SonarQube.
2. **Docker Integration**
   - Building Docker images.
   - Pushing images to Amazon Elastic Container Registry (ECR).
3. **Deployment to AWS**
   - Deploying Docker images to Amazon ECS with AWS Fargate.

---

## Jenkins Configuration and GitHub Integration
- **Setting up Jenkins**: Configuring GitHub credentials and defining pipeline stages using a `Jenkinsfile`.
- **Running Tests**: Unit tests with npm and code quality analysis with SonarQube.

---

## Docker Integration and Deployment
- **Building Docker Images**: Creating lightweight Docker images using Alpine.
- **Vulnerability Scanning**: Using Trivy to scan Docker images.
- **AWS Integration**:
  - Pushing Docker images to AWS ECR.
  - Deploying to AWS ECS using Fargate.

---

## AWS ECS Deployment
1. **Infrastructure Setup**:
   - Creating ECS clusters and services.
   - Configuring load balancers and security groups.
2. **Application Management**:
   - Deploying and managing containerized applications.
   - Monitoring performance and ensuring public accessibility.
3. **Cost Optimization**:
   - Deleting unused resources to prevent unnecessary expenses.

---

## Set up Jenkins 
Run the command below to create the Jenkins Container:
```sh
docker run -d --name jenkins-dind \
-p 8080:8080 -p 50000:50000 \
-v /var/run/docker.sock:/var/run/docker.sock \
-v $(which docker):/usr/bin/docker \
-u root \
-e DOCKER_GID=$(getent group docker | cut -d: -f3) \
jenkins/jenkins:lts
```

Check Running Docker Container
```sh
docker ps
```

Log into the Jenkins container
```sh
docker exec -it jenkins-dind bash
```
Run the following bash commands in the Jenkins Container's Terminal:
```sh
groupadd -for -g $DOCKER_GID docker
usermod -aG docker jenkins
exit
```
```sh
docker restart jenkins-dind
```

Check the Jenkins Container Logs to Get Jenkins' Initial Admin Password
```sh
docker logs -f jenkins-dind
```

On your browser, open:
```sh 
PublicIP:8080
```
Paste Initial Admin Password & Install Suggested Plugins
Create an Admin User Account with username & password, email, etc.


## Configure Jenkins & GitHub Integration

***Create a Classic Personal Access Token on GitHub***
On your GitHub Account, 

   1. Click on the User Account
   2. Click on Settings
   3. Developer settings, and select Personal access tokens and Click Tokens (classic)
   4. Generate new token, and select Generate new token (classic)
         Note: jenkins-git-dind, Expiration: 90 days, and
               Scopes (select the following):
                  repo
                  admin:repo_hook (For Webhooks)
         Generate token & save it somewhere safe

***Add the GitHub Personal Access Token to Jenkins Credentials***
On the Jenkins UI,  

   1. Click on Manage Jenkins
   2. Click Credentials
   3. Under System, click global, and Add Credentials
   4. Select Kind: "Username with password", Scope: Global, Password: "Paste the jenkins-git-dind here", ID: "jen-git-dind", and Description: "jen-git-dind". Create.

***Create Jenkinsfile (a.k.a Pipeline As A Code)***


## Set up Sonarqube 

***Set the following Recommended Values***
Change your user to root first,
```sh
sudo su
```
or
```sh
 sudo i
```
Run the commands:
```sh
sysctl -w vm.max_map_count=524288
sysctl -w fs.file-max=131072
ulimit -n 131072
ulimit -u 8192
```
Exit the Root User:
```sh
exit
```

Run the command below to create the SonarQube Container (Give it a few minutes to get ready):
```sh
docker run -d --name sonarqube-dind \
-e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
-p 9000:9000 \
sonarqube:latest
```

Check the logs of the SonarQube Container:
```sh
docker logs -f sonarqube-dind
```

On your browser, open:
```sh 
PublicIP:9000
```
Username: admin
Password: admin
Update your password by providing a new password

Create a Local Project in SonarQube with Project key, GitHub branch (main), and "Use the global setting".

Create a SonarQube Token and Add it to Jenkins Credentials:
   1. Click on the User Account, and click on "My Account"
   2. Go to Security, create a token of type "Global Analysis Token", and expiry date
   3. Copy and Save the token somewhere safe.
   4. Go to Jenkins Credentials, select Kind "Secret text", Paste sonar token as the secret and provide ID and description

Go to Jenkins > Manage Jenkins > Systems > SonarQube Installation, and add the SonarQube URL (http://PublicIP:9000) and select the token. Apply and Save. 

Go to Jenkins > Manage Jenkins > Add SonarQube Scanner (Install Automatically). Save and Apply


***Put Both Jenkins and SonarQube Containers on the same Docker Network***
```sh
docker network ls
```
Create Docker Network: 
```sh
docker network create dind-network
```
Add Jenkins and SonarQube Containers to dind-network:
```sh
docker network connect dind-network jenkins-dind
```
```sh
docker network connect dind-network sonarqube-dind
```

***Login to Jenkins Container & Establish Communication to the SonarQube Container***
```sh
docker exec -it jenkins-dind bash
```
Update & Install Ping in Jenkins Container once you've logged in:
```sh
apt-get update
apt-get install unzip curl iputils-ping -y
```

Use Jenkins Container bash to Ping SonarQube Container:
```sh
docker exec -it jenkins-dind ping sonarqube-dind
```
You should be seeing bytes of data coming in showing established connection between the 2 containers.
Exit the Jenkins Container:
```sh
exit
```


## Set up Trivy
Login to the Jenkins Container:
```sh
docker exec -it jenkins-dind bash
```
Check if Trivy is installed in the Jenkins Container:
```sh
trivy --version
```
```sh
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sudo sh -s -- -b /usr/local/bin v0.62.1
```
Check if Trivy Installation was a success & exit: 
```sh
trivy --version
exit
```

Restart the Jenkins Container (optional but recommended):
```sh
docker restart jenkins-dind
```

## Set up AWS CLI & AWS IAM USER
Log in to the Jenkins Container:
```sh
docker exec -it jenkins-dind bash
```
```sh
apt-get update -y
apt-get install unzip curl -y
```
```sh
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install
```
Check AWS CLI version
```sh
aws --version
```

Go to AWS Mgt Console > IAM > Create IAM User (Attach AmazonEC2ContainerRegistryFullAccess policy)

Go to the IAM User > Click Security credentials > Create Access Keys (with CLI use-case) > Create & Download Access keys. 

On your terminal, login to Jenkins Container
```sh
docker exec -it jenkins-dind bash
```

***Configure AWS IAM User***
```sh
aws configure
```
Copy and Paste Access key ID, Secret Access key, Region, etc. and exit

Note: Add Access key ID & Secret Access key to Jenkins credentials as Kind: AWS Credentials.


***Create Amazon ECR Repository on AWS Mgt Console, and that should provide the links to login, tag, and push image to ECR***

***Next, create Amazon ECS Cluster (Fargate) on AWS Mgt Console, to serve the docker image pushed to Amazon ECR***
1. Create cluster > Select Fargate
2. Click on Task definition on the left panel and create a new one, and add the instance type, task role, task execution role, container details, etc., and create
3. Go into the ECS cluster created earlier and create a Service to expose the container to the public internet


***Jenkins Plugins Needed*** 
Install the following plugins:
   1. SonarQube Scanner 
   2. Sonar Quality Gates
   3. Pipeline
   4. Docker pipeline
   5. Docker plugin
   6. AWS SDK


## Resource Cleanup
To maintain cost efficiency, ensure proper deletion of:
- Target groups
- Load balancers
- ECS clusters
- Any associated AWS resources.

---

## Conclusion
This guide empowers developers with a streamlined CI/CD pipeline setup. By combining Jenkins, Docker, and AWS ECS, you can build, test, and deploy applications effectively while maintaining high-quality standards and cost efficiency.

---
