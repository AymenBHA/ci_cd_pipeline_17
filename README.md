# In this project, we will build a complete CI/CD pipeline using Forgejo (Git), Jenkins, and Docker.

How It Works:
VM 1 (CI/CD Server): Hosts Forgejo (Git server), Jenkins (automation), and Docker Registry (image storage).
VM 2 (Production Server): Runs the deployed application pulled from the registry.
The Pipeline: Code push triggers webhook -> Jenkins builds Docker image -> Pushes to registry -> Deploys to VM 2.
Key Components:

Forgejo: Self-hosted Git server with webhook support
Jenkins: Automation server that orchestrates the CI/CD pipeline
Docker: Containerization for consistent deployments
Nginx: Reverse proxy for routing traffic to services

<br><br>
<!-- Forced Line Break -->
<br><br>

## Goal: Provision and configure the first EC2 instance that will host Forgejo (Git server), Jenkins (automation), and Docker Registry (image storage).

### Step 1: Launch EC2 Instance
Navigate to AWS Console -> EC2 -> Instances -> Click Launch instances
Configure the instance:
Name: vm-cicd (exactly as shown)
Application and OS Images (AMI): Select Ubuntu Server 24.04 LTS
Instance type: t3.micro
Key pair: Create a new key pair or select an existing one (you'll need this to SSH)
Network settings: Click Edit and configure:
Firewall (security groups): Create security group
Add these inbound rules:
SSH (Port 22): Source = 0.0.0.0/0
HTTP (Port 80): Source = 0.0.0.0/0
Custom TCP (Port 2222): Source = 0.0.0.0/0
Configure storage: Change to 30 GiB gp3
Click Launch instance
Wait for the instance state to show Running

### Step 2: Connect to our EC2 Instance 
Using SSH from your terminal:
ssh -i /path/to/your-key.pem ubuntu@<VM1_PUBLIC_IP>


### Step 3: Create Swap File (Prevents Jenkins from Crashing)
Jenkins requires significant memory. A swap file acts as overflow memory when RAM is full.
Run these commands one by one:
```bash
# Create a 4GB swap file
sudo fallocate -l 4G /swapfile

# Secure the swap file (only root can read/write)
sudo chmod 600 /swapfile

# Mark the file as swap space
sudo mkswap /swapfile
# Enable the swap file
sudo swapon /swapfile

# Make swap permanent (survives reboots)
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

Verify swap is active:

free -h
```
You should see 4GB under the "Swap" row.

### Step 4: Install Docker & Docker Compose
Docker will run our containers (Forgejo, Jenkins, Nginx).
```bash
# Update package list and install Docker
sudo apt update && sudo apt install -y docker.io docker-compose-v2

# Add ubuntu user to docker group (allows running docker without sudo)
sudo usermod -aG docker ubuntu

# Apply group membership without logging out
newgrp docker

# Allow Jenkins container to access host Docker
sudo chmod 666 /var/run/docker.sock

Verify Docker is working:

docker --version
docker compose version
```

### Step 5: Configure Insecure Registry
We're running a private Docker registry on port 80 (not HTTPS). Docker needs to be told this is allowed.
Important: Replace <VM1_PUBLIC_IP> with your actual VM 1 public IP address.
```bash
echo '{"insecure-registries": ["<VM1_PUBLIC_IP>:80"]}' | sudo tee /etc/docker/daemon.json

Restart Docker to apply changes:

sudo systemctl restart docker
```

### Step 6: Generate SSH Keys (For VM 1 to VM 2 Communication)
Jenkins on VM 1 will need to SSH into VM 2 to deploy the application.
```bash
ssh-keygen -t rsa -b 4096
```
Press Enter three times to:

Accept default file location
Skip passphrase (first prompt)
Skip passphrase confirmation (second prompt)
View your public key (you'll need this later):
```bash
cat ~/.ssh/id_rsa.pub
```

the fundamental difference between Continuous Delivery and Continuous Deployment is that delivery requires manual approval for release, deployment is fully automated

<br><br>
<!-- Forced Line Break -->
<br><br>

## Goal: Deploy Forgejo (self-hosted Git server), Jenkins, and Nginx using Docker Compose, then configure Forgejo with a repository and webhook.

### Step 1: Create Project Directory and Environment File
Still connected to VM 1, create a directory for our CI/CD stack:
```bash
mkdir ~/cicd-stack && cd ~/cicd-stack

Create the environment file:

nano .env
```
Paste this content (replace <VM1_PUBLIC_IP> with your actual VM 1 public IP):

EC2_IP=<VM1_PUBLIC_IP>

Save and exit: Press Ctrl+X, then Y, then Enter

### Step 2: Create Nginx Configuration
Nginx will act as a reverse proxy, routing traffic to Forgejo and Jenkins.
```bash

nano nginx.conf
```
Paste this complete configuration:
```yaml

server {
    listen 80;
    server_name _;
    client_max_body_size 0;

    location /jenkins {
        proxy_pass http://jenkins:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
    }

    location / {
        proxy_pass http://forgejo:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header Origin http://$host;
    }
}
```
Save and exit: Ctrl+X, Y, Enter

What this does:
Requests to /jenkins go to Jenkins container
All other requests go to Forgejo container
Headers are forwarded so applications know the original request details

### Step 3: Create Docker Compose File
This file defines all our services (Nginx, Forgejo, Jenkins).
```bash

nano docker-compose.yml
```
Paste this complete configuration:

```yaml

services:
  nginx:
    image: nginx:stable-alpine
    container_name: nginx-proxy
    ports: ["80:80"]
    volumes: ["./nginx.conf:/etc/nginx/conf.d/default.conf:ro"]
    networks: ["cicd-net"]

  forgejo:
    image: codeberg.org/forgejo/forgejo:15
    container_name: forgejo
    ports: ["2222:22"]
    environment:
      - FORGEJO__server__ROOT_URL=http://${EC2_IP}/
      - FORGEJO__webhook__ALLOWED_HOST_LIST=${EC2_IP},jenkins,loopback
      - FORGEJO__packages__ENABLED=true
      - FORGEJO__packages__CONTAINER_REGISTRY_ENABLED=true
      - FORGEJO__security__CSRF_TRUSTED_ORIGINS=http://${EC2_IP}
    networks: ["cicd-net"]
    volumes: ["./forgejo-data:/data"]

  jenkins:
    build:
      context: .
      dockerfile_inline: |
        FROM jenkins/jenkins:2.516.1-alpine-jdk21
        USER root
        RUN apk add --update docker-cli
        RUN mkdir -p /usr/share/jenkins/ref/init.groovy.d
        RUN echo 'import jenkins.model.*; import hudson.security.*; def instance = Jenkins.getInstance(); instance.setAuthorizationStrategy(new AuthorizationStrategy.Unsecured()); instance.setSecurityRealm(new HudsonPrivateSecurityRealm(false)); instance.save()' > /usr/share/jenkins/ref/init.groovy.d/disable-auth.groovy
        USER jenkins
    container_name: jenkins
    restart: always
    user: root
    environment:
      - JAVA_OPTS=-Xmx512m -Xms256m -Djenkins.install.runSetupWizard=true
      - JENKINS_OPTS=--prefix=/jenkins
    networks: ["cicd-net"]
    volumes:
      - ./jenkins-data:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    deploy:
      resources:
        limits:
          memory: 1G

networks:
  cicd-net:
    name: cicd-net

Save and exit: Ctrl+X, Y, Enter
```

### Step 4: Start the Stack
Launch all containers:
```bash
docker compose up -d
```

This will:

Build the custom Jenkins image (takes 2-3 minutes)
Download Forgejo and Nginx images
Start all three containers
Check that all containers are running:
```bash
docker ps
```

we should see 3 containers: nginx-proxy, forgejo, and jenkins

### Step 5: Configure Forgejo
Access Forgejo: Open your browser and go to http://<VM1_PUBLIC_IP>/

Initial Setup:

You'll see a setup page
Leave all defaults as-is
Scroll to the bottom and click Administrator account settings dropdown
Register Your Account:

Choose a username (remember this!)
Enter an email (can be fake: user@example.com)
Set a password
Click Install Forgejo

Create a Repository:

Click the + icon (top right)
Select New Repository
Repository Name: Choose a name (e.g., devspace, portfolio, myapp)
Visibility: Private or Public (your choice)
Leave other options as default
Click Create Repository

Add Webhook:
Inside your repository, click Settings (top right)
Click Webhooks in the left sidebar
Click Webhooks -> Add webhook -> Forgejo
Configure:
Target URL: http://<VM1_PUBLIC_IP>/jenkins/generic-webhook-trigger/invoke?token=devspace-token
HTTP Method: POST
Post Content Type: application/json
Trigger On: Push events (should be checked by default)
Click Add Webhook

<br><br>
<!-- Forced Line Break -->
<br><br>

## Goal: Configure Jenkins to listen for webhook events from Forgejo and create a pipeline.

### Step 1: Access Jenkins
Open your browser and go to: http://<VM1_PUBLIC_IP>/jenkins
Install Plugins:
Select Install suggested plugins
Wait for installation to complete

Create Admin User:
Fill in the form (or click "Skip and continue as admin")
Click Save and Continue
Click Save and Finish
Click Start using Jenkins

## Step 2: Install Required Plugins
From Jenkins Dashboard, click Manage Jenkins (top right settings icon beside Sign in)
Click Plugins
Click Available plugins tab

Search for and install these plugins:
Generic Webhook Trigger (for Forgejo integration)
Docker Pipeline (for building Docker images)
SSH Agent (for deploying to VM 2)
Check the box next to each plugin

Click Install (bottom right)
Check Restart Jenkins when installation is complete
Wait for Jenkins to restart (refresh the page after 30 seconds)

### Step 3: Create Jenkins Credentials
We need to store credentials for:
SSH access to VM 2
Forgejo registry access

A. SSH Credential for VM 2:
Go to Manage Jenkins -> Credentials
Click (global) domain
Click the prompt at the center (How about adding some credentials?)
Configure:
Kind: SSH Username with private key
ID: vm2-ssh (you'll use this in Jenkinsfile)
Username: ubuntu
Private Key: Click Enter directly

Get the private key from VM 1:
cat ~/.ssh/id_rsa

Copy the entire output (including -----BEGIN and -----END lines)
Paste into Jenkins
Click Create
B. Forgejo Registry Credential:
Click + Add Credentials
Configure:
Kind: Username with password
Username: Your Forgejo username
Password: Your Forgejo password
ID: forgejo-registry (you'll use this in Jenkinsfile)
Click Create
### Step 4: Create Pipeline
Go back to Jenkins Dashboard

Click New Item (left sidebar)

Configure:

Name: devspace-pipeline (or match your repo name)
Type: Select Pipeline
Click OK
Configure Triggers:

Scroll down to Triggers section
Check Generic Webhook Trigger
In the Token field, enter: devspace-token
Critical: This must exactly match the token in your Forgejo webhook URL
Configure Pipeline:

Scroll down to Pipeline section
Definition: Pipeline script from SCM
SCM: Git
Repository URL: Get this from Forgejo:
Go to your Forgejo repository
Click the HTTP button
Copy the URL (e.g., http://<VM1_IP>/username/reponame.git)
Paste into Jenkins
Credentials: Select your Forgejo credential
Branch Specifier: */master (or */main if that's your default branch)
Script Path: Jenkinsfile
Click Save

### Step 5: Verify Webhook Connection
The webhook handshake is now complete:

Forgejo sends events to: http://<VM1_IP>/jenkins/generic-webhook-trigger/invoke?token=devspace-token
Jenkins listens for token: devspace-token
When you push code to Forgejo, the webhook will automatically trigger the Jenkins pipeline.

<br><br>
<!-- Forced Line Break -->
<br><br>

## Goal: Provision and configure the second EC2 instance that will run our deployed application.

### Step 1: Launch EC2 Instance
Navigate to AWS Console -> EC2 -> Instances -> Click Launch instances
Configure the instance:
Name: vm-web (exactly as shown)
Application and OS Images (AMI): Select Ubuntu Server 24.04 LTS
Instance type: t3.micro
Key pair: Use the same key pair as VM 1
Network settings: Click Edit and configure:
Firewall (security groups): Create security group
Add these inbound rules:
SSH (Port 22): Source = 0.0.0.0/0
HTTP (Port 80): Source = 0.0.0.0/0
Configure storage: Change to 30 GiB gp3
Click Launch instance
Wait for the instance state to show Running
Copy the Public IPv4 address - you'll need this for deployment
Step 2: Connect to VM 2
Using EC2 Instance Connect:

Select your vm-web instance
Click Connect button
Choose EC2 Instance Connect tab
Click Connect
Step 3: Install Docker
Run these commands on VM 2:
```bash
# Update package list and install Docker
sudo apt update && sudo apt install -y docker.io docker-compose-v2

# Add ubuntu user to docker group
sudo usermod -aG docker ubuntu

# Apply group membership
newgrp docker

Verify Docker is working:

docker --version
```

### Step 4: Configure Insecure Registry
VM 2 needs to pull images from VM 1's registry.

Important: Replace <VM1_PUBLIC_IP> with VM 1's actual public IP address.

echo '{"insecure-registries": ["<VM1_PUBLIC_IP>:80"]}' | sudo tee /etc/docker/daemon.json

Restart Docker:

sudo systemctl restart docker

### Step 5: Authorize Jenkins SSH Access
Jenkins on VM 1 needs passwordless SSH access to VM 2 for deployment.

On VM 1, get the public key:

cat ~/.ssh/id_rsa.pub

Copy the entire output.

On VM 2, add the key to authorized_keys:
```bash

# Create .ssh directory if it doesn't exist
mkdir -p ~/.ssh

# Open the authorized_keys file
nano ~/.ssh/authorized_keys
```
Paste VM 1's public key on a new line.

Save and exit: Ctrl+X, Y, Enter

Secure the file:
```bash

chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```
Test the connection from VM 1:
```bash

ssh ubuntu@<VM2_PUBLIC_IP>
```
You should connect without a password prompt. Type exit to return to VM 1.

### Step 6: Prepare Application Directory
On VM 2, create the directory where your app will run:
```bash

mkdir ~/devspace && cd ~/devspace
```
Create the docker-compose file:
```bash

nano docker-compose.yml
```
Paste this configuration (replace placeholders with your actual values):

```bash

services:
  portfolio:
    image: <VM1_PUBLIC_IP>:80/<FORGEJO_USERNAME>/<REPO_NAME>:latest
    container_name: kodekloud-portfolio-app
    ports:
      - "80:80"
    restart: unless-stopped

```
Example:
If VM1 IP is 54.123.45.67, username is john, and repo is devspace:

image: 54.123.45.67:80/john/devspace:latest

Save and exit: Ctrl+X, Y, Enter

What this does:

Pulls the Docker image from VM 1's registry
Runs it on port 80
Automatically restarts if it crashes
Checkpoint:

EC2 instance named vm-web is running
Docker installed and configured
Insecure registry configured to trust VM 1
SSH access from VM 1 to VM 2 working
Application directory and docker-compose.yml created
Click Check to validate your setup before proceeding.

<br><br>
<!-- Forced Line Break -->
<br><br>

## Goal: Configure your KodeKloud terminal as a development environment and connect it to Forgejo.

### Step 1: Configure Git Identity
Git needs to know who is making commits. Run these commands in your KodeKloud terminal:

```bash
git config --global user.name "Developer"
git config --global user.email "dev@example.com"
```
Verify the configuration:
```bash
git config --global --list
```

### Step 2: Generate SSH Key
Create an SSH key pair for authenticating with Forgejo:
```bash
ssh-keygen -t rsa -b 4096
```
Press Enter three times to:
Accept default file location (~/.ssh/id_rsa)
Skip passphrase (first prompt)
Skip passphrase confirmation (second prompt)
Display your public key:
```bash
cat ~/.ssh/id_rsa.pub
```
Copy the entire output - you'll need this in the next step.

### Step 3: Add SSH Key to Forgejo
Go to Forgejo UI: http://<VM1_PUBLIC_IP>/
Log in with your account
Click your Profile Picture (top right corner)
Select Settings
In the left sidebar, click SSH / GPG Keys
Click Add Key button on Manage SSH keys
Configure:
Key Name: KodeKloud Terminal (or any descriptive name)
Content: Paste the public key you copied
Click Add Key
You should see your key listed.

### Step 4: Initialize Git Repository
Navigate to your project directory (adjust path if needed):
```bash
cd ~/devspace
```

If the directory doesn't exist, create it:
```bash
mkdir -p ~/devspace && cd ~/devspace
```

Initialize Git:
```bash
git init
```

Add Forgejo as the remote origin:
```bash
git remote add origin ssh://git@<VM1_PUBLIC_IP>:2222/<YOUR_USERNAME>/<YOUR_REPO>.git
```

Important Notes:
Replace <VM1_PUBLIC_IP> with VM 1's public IP
Replace <YOUR_USERNAME> with your Forgejo username
Replace <YOUR_REPO> with your repository name
Don't forget port 2222 - this is Forgejo's SSH port
Example:
```bash
git remote add origin ssh://git@54.123.45.67:2222/john/devspace.git
```

Verify the remote:
```bash
git remote -v
```

### Step 5: Test SSH Connection
Test that you can connect to Forgejo via SSH:
```bash
ssh -T git@<VM1_PUBLIC_IP> -p 2222
```
You may see a message about host authenticity. Type yes and press Enter.

You should see a message like:
Hi there, <username>! You've successfully authenticated...
If you see this, your SSH connection is working!

<br><br>
<!-- Forced Line Break -->
<br><br>


## Goal: Create the Jenkinsfile, commit your code, and trigger the automated CI/CD pipeline.

### Step 1: Update the Jenkinsfile
The Jenkinsfile defines your CI/CD pipeline. In your KodeKloud terminal:
```bash
cd ~/devspace
sudo apt update && sudo apt install nano -y
nano Jenkinsfile
```
Update the environment variables after entering nano:

```yaml

pipeline {
    agent any

    environment {
        REGISTRY = "<VM1_PUBLIC_IP>:80"              // VM 1's public IP with port 80
        IMAGE_NAME = "<USERNAME>/<REPO_NAME>"        // Your Forgejo username and repo name
        VM2_IP = "<VM2_PUBLIC_IP>"                   // VM 2's public IP
        VM2_SSH_CRED_ID = "vm2-ssh"                  // Jenkins credential ID for SSH
        FORGEJO_CRED_ID = "forgejo-registry"         // Jenkins credential ID for registry
    }

```
Update these values:

REGISTRY: VM 1's public IP with :80 (e.g., 54.123.45.67:80)
IMAGE_NAME: Your Forgejo username and repo (e.g., john/devspace)
VM2_IP: VM 2's public IP (e.g., 54.98.76.54)
VM2_SSH_CRED_ID: Should be vm2-ssh (the credential ID you created in Jenkins)
FORGEJO_CRED_ID: Should be forgejo-registry (the credential ID you created in Jenkins)
Save and exit: Ctrl+X, Y, Enter

### Step 2: Update Profile Picture
If you want to add a GitHub profile picture:
```bash
curl -s https://api.github.com/users/<YOUR_GITHUB_USERNAME> | grep -o 'https://avatars.githubusercontent.com/u/[^"]*' | xargs curl -o profile_picture.jpg
```

Replace <YOUR_GITHUB_USERNAME> with your actual GitHub username.

### Step 3: Update the Config.js
```bash

cd ~/devspace
nano config.js
```

```javascript
const PORTFOLIO_CONFIG = {
    // Basic Information
    name: "John Doe",
    role: "DevOps Engineer",
    profileImage: "https://via.placeholder.com/250/0d162d/00e5ff?text=Profile+Photo", // Local path like "profile.jpg" or a URL

    // Hero Section
    heroSubtitle: "Building scalable, resilient, and secure architectures in the cloud.",

    // About Me Section
    // You can add multiple paragraphs here
    aboutMeParagraphs: [
        "I am a passionate <strong>DevOps Engineer</strong> with a deep love for automation, cloud infrastructure, and continuous integration/continuous deployment. Just like exploring the vastness of space, I enjoy navigating complex systems and bringing order to chaotic environments.",
        "My mission is to bridge the gap between development and operations, ensuring smooth sailing for code from the developer's machine all the way to the production universe. I specialize in Docker, Kubernetes, AWS, and modern CI/CD pipelines."
    ],

    // Stats in About Section
    stats: [
        { number: "50+", label: "Deployments" },
        { number: "99.9%", label: "Uptime" },
        { number: "100%", label: "Automated" }
    ]
};
```

Save and exit: Ctrl+X, Y, Enter

### Step 4: Commit and Push
Add all files to Git:
```bash
git add .
```

Commit your changes:
```bash
git commit -m "feat: complete end-to-end CI/CD setup"
```

Push to Forgejo (this triggers the pipeline!):
```bash
git push -u origin master
```

If your default branch is main instead of master:
```bash
git branch -M main
git push -u origin main
```

### Step 5: Monitor the Pipeline
Watch the Push: After pushing, the webhook should trigger immediately

Check Jenkins:

Go to http://<VM1_PUBLIC_IP>/jenkins
You should see your pipeline running
Click on the build number (e.g., #1)
Click Console Output to watch the logs in real-time

Pipeline Stages:

Build: Creates Docker image from your code
Push to Registry: Uploads image to Forgejo's registry
Deploy to VM2: SSH into VM 2 and runs the container
Wait for Completion: The pipeline takes 2-3 minutes

### Step 6: Verify Deployment
Once the pipeline shows SUCCESS:

Open your browser
Go to: http://<VM2_PUBLIC_IP>/
You should see your deployed application!
Troubleshooting:

If you see "Connection refused": Wait 30 seconds for the container to start
If you see "502 Bad Gateway": Check Jenkins console output for errors
If the pipeline fails: Read the error message in Jenkins console output
Watch Jenkins automatically:

Detect the push
Build the new image
Deploy to VM 2
Refresh http://<VM2_PUBLIC_IP>/ to see your changes

Checkpoint:

Jenkinsfile created with correct environment variables
Code committed and pushed to Forgejo
Webhook triggered Jenkins pipeline automatically
Pipeline completed all stages successfully
Application accessible at http://<VM2_PUBLIC_IP>/
Automated deployment working on subsequent pushes
Validate both EC2 instances are running for deployment.

<br><br>
<!-- Forced Line Break -->
<br><br>

What we've Built:
Self-Hosted Git Server: Forgejo running on VM 1 with webhook integration
Automation Server: Jenkins configured to listen for Git events and orchestrate deployments
Container Registry: Docker registry hosted on VM 1 for storing application images
Production Environment: VM 2 running your containerized application
Automated Pipeline: Code push -> Build -> Test -> Deploy workflow

Key Concepts Implemented:
Webhook Integration: Automatic pipeline triggering on code push
Containerization: Docker for consistent application packaging
Registry Management: Private Docker registry for image storage
SSH Automation: Passwordless deployment to production servers
Reverse Proxy: Nginx routing traffic to multiple services
Infrastructure as Code: Docker Compose for reproducible deployments


Real-World Applications:
Automated software delivery pipelines
Microservices deployment workflows
Development to production automation
Team collaboration with Git workflows
Container orchestration and management


