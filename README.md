# CI/CD Pipeline with Forgejo, Jenkins, and Docker


This project outlines how to build a complete CI/CD pipeline using Forgejo (Git), Jenkins, and Docker.

### How It Works
*   **VM 1 (CI/CD Server):** Hosts Forgejo (Git server), Jenkins (automation), and Docker Registry (image storage).
*   **VM 2 (Production Server):** Runs the deployed application pulled from the registry.

The pipeline is initiated by a code push, which triggers a webhook. Jenkins then builds a Docker image, pushes it to the private registry, and deploys it to the production server (VM 2).

### Key Components
*   **Forgejo:** Self-hosted Git server with webhook support.
*   **Jenkins:** Automation server that orchestrates the CI/CD pipeline.
*   **Docker:** Containerization for consistent application deployments.
*   **Nginx:** Reverse proxy for routing traffic to various services.

---

## Goal: Provision and configure the first EC2 instance that will host Forgejo (Git server), Jenkins (automation), and Docker Registry (image storage).

### Step 1: Launch EC2 Instance
Navigate to AWS Console -> EC2 -> Instances -> Click **Launch instances**.
Configure the instance:
*   **Name:** `vm-cicd` (exactly as shown)
*   **Application and OS Images (AMI):** Select Ubuntu Server 24.04 LTS
*   **Instance type:** `t3.micro`
*   **Key pair:** Create a new key pair or select an existing one (you'll need this to SSH).
*   **Network settings:** Click **Edit** and configure:
    *   **Firewall (security groups):** Create security group
    *   Add these inbound rules:
        *   SSH (Port 22): Source = `0.0.0.0/0`
        *   HTTP (Port 80): Source = `0.0.0.0/0`
        *   Custom TCP (Port 2222): Source = `0.0.0.0/0`
*   **Configure storage:** Change to `30 GiB gp3`.
*   Click **Launch instance** and wait for the instance state to show **Running**.

### Step 2: Connect to our EC2 Instance
Using SSH from your terminal:
```bash
ssh -i /path/to/your-key.pem ubuntu@<VM1_PUBLIC_IP>
```

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
```
Verify swap is active:
```bash
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
```
Verify Docker is working:
```bash
docker --version
docker compose version
```

### Step 5: Configure Insecure Registry
We're running a private Docker registry on port 80 (not HTTPS). Docker needs to be told this is allowed.
**Important:** Replace `<VM1_PUBLIC_IP>` with your actual VM 1 public IP address.
```bash
echo '{"insecure-registries": ["<VM1_PUBLIC_IP>:80"]}' | sudo tee /etc/docker/daemon.json
```
Restart Docker to apply changes:
```bash
sudo systemctl restart docker
```

### Step 6: Generate SSH Keys (For VM 1 to VM 2 Communication)
Jenkins on VM 1 will need to SSH into VM 2 to deploy the application.
```bash
ssh-keygen -t rsa -b 4096
```
Press Enter three times to accept defaults and skip creating a passphrase.

View your public key (you'll need this later):
```bash
cat ~/.ssh/id_rsa.pub
```
The fundamental difference between Continuous Delivery and Continuous Deployment is that delivery requires manual approval for release, while deployment is fully automated.

---

## Goal: Deploy Forgejo (self-hosted Git server), Jenkins, and Nginx using Docker Compose, then configure Forgejo with a repository and webhook.

### Step 1: Create Project Directory and Environment File
Still connected to VM 1, create a directory for our CI/CD stack:
```bash
mkdir ~/cicd-stack && cd ~/cicd-stack
```
Create the environment file:
```bash
nano .env
```
Paste this content (replace `<VM1_PUBLIC_IP>` with your actual VM 1 public IP):
```
EC2_IP=<VM1_PUBLIC_IP>
```
Save and exit: Press `Ctrl+X`, then `Y`, then `Enter`.

### Step 2: Create Nginx Configuration
Nginx will act as a reverse proxy, routing traffic to Forgejo and Jenkins.
```bash
nano nginx.conf
```
Paste this complete configuration:
```nginx
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
Save and exit: `Ctrl+X`, `Y`, `Enter`.

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
```
Save and exit: `Ctrl+X`, `Y`, `Enter`.

### Step 4: Start the Stack
Launch all containers:
```bash
docker compose up -d
```
This will build the Jenkins image, download the others, and start all containers. Check the status:
```bash
docker ps
```
You should see three running containers: `nginx-proxy`, `forgejo`, and `jenkins`.

### Step 5: Configure Forgejo
Access Forgejo in your browser at `http://<VM1_PUBLIC_IP>/`.

**Initial Setup:**
1.  On the setup page, leave all defaults as-is.
2.  Scroll to the bottom and expand **Administrator account settings**.
3.  Choose a username, enter an email (e.g., `user@example.com`), and set a password.
4.  Click **Install Forgejo**.

**Create a Repository:**
1.  Click the **+** icon (top right) and select **New Repository**.
2.  Enter a **Repository Name** (e.g., `devspace`).
3.  Choose the visibility and click **Create Repository**.

**Add Webhook:**
1.  Inside your repository, go to **Settings** -> **Webhooks**.
2.  Click **Add webhook** -> **Forgejo**.
3.  **Target URL:** `http://<VM1_PUBLIC_IP>/jenkins/generic-webhook-trigger/invoke?token=devspace-token`
4.  **HTTP Method:** `POST`
5.  **Post Content Type:** `application/json`
6.  Ensure **Push events** is checked under **Trigger On**.
7.  Click **Add Webhook**.

---

## Goal: Configure Jenkins to listen for webhook events from Forgejo and create a pipeline.

### Step 1: Access Jenkins
Go to `http://<VM1_PUBLIC_IP>/jenkins` in your browser.
1.  Select **Install suggested plugins** and wait for completion.
2.  Create an admin user or click **Skip and continue as admin**.
3.  Click **Save and Finish**, then **Start using Jenkins**.

### Step 2: Install Required Plugins
1.  From the Jenkins Dashboard, go to **Manage Jenkins** -> **Plugins**.
2.  In the **Available plugins** tab, search for and select:
    *   `Generic Webhook Trigger`
    *   `Docker Pipeline`
    *   `SSH Agent`
3.  Click **Install** and check the box to **Restart Jenkins when installation is complete**.

### Step 3: Create Jenkins Credentials
Go to **Manage Jenkins** -> **Credentials** -> **(global)**.

**A. SSH Credential for VM 2:**
1.  Click **+ Add Credentials**.
2.  **Kind:** SSH Username with private key
3.  **ID:** `vm2-ssh`
4.  **Username:** `ubuntu`
5.  **Private Key:** Select **Enter directly**.
6.  Get the private key from VM 1: `cat ~/.ssh/id_rsa`. Copy the entire output (including `-----BEGIN...` and `-----END...`) and paste it into the key field in Jenkins.
7.  Click **Create**.

**B. Forgejo Registry Credential:**
1.  Click **+ Add Credentials**.
2.  **Kind:** Username with password
3.  **Username:** Your Forgejo username
4.  **Password:** Your Forgejo password
5.  **ID:** `forgejo-registry`
6.  Click **Create**.

### Step 4: Create Pipeline
1.  From the Jenkins Dashboard, click **New Item**.
2.  **Name:** `devspace-pipeline` (or match your repo name).
3.  **Type:** Select **Pipeline** and click **OK**.
4.  **Triggers:** Check **Generic Webhook Trigger** and set the **Token** to `devspace-token`. This must match the token in the Forgejo webhook URL.
5.  **Pipeline:**
    *   **Definition:** Pipeline script from SCM
    *   **SCM:** Git
    *   **Repository URL:** Copy the HTTP URL from your Forgejo repository (e.g., `http://<VM1_IP>/username/reponame.git`).
    *   **Credentials:** Select your `forgejo-registry` credential.
    *   **Branch Specifier:** `*/master` (or `*/main`).
    *   **Script Path:** `Jenkinsfile`
6.  Click **Save**.

### Step 5: Verify Webhook Connection
The connection is now established. When you push code to Forgejo, the webhook will automatically trigger the Jenkins pipeline.

---

## Goal: Provision and configure the second EC2 instance that will run our deployed application.

### Step 1: Launch EC2 Instance
1.  Navigate to AWS Console -> EC2 -> Instances -> Click **Launch instances**.
2.  **Name:** `vm-web` (exactly as shown)
3.  **AMI:** Ubuntu Server 24.04 LTS
4.  **Instance type:** `t3.micro`
5.  **Key pair:** Use the same key pair as VM 1.
6.  **Network settings:** Create a new security group with these inbound rules:
    *   SSH (Port 22): Source = `0.0.0.0/0`
    *   HTTP (Port 80): Source = `0.0.0.0/0`
7.  **Storage:** `30 GiB gp3`
8.  Click **Launch instance** and copy the Public IPv4 address.

### Step 2: Connect to VM 2 and Install Docker
Connect to `vm-web` using your preferred method (e.g., EC2 Instance Connect).
```bash
# Update package list and install Docker
sudo apt update && sudo apt install -y docker.io docker-compose-v2

# Add ubuntu user to docker group
sudo usermod -aG docker ubuntu

# Apply group membership
newgrp docker

# Verify Docker
docker --version
```

### Step 4: Configure Insecure Registry
VM 2 needs to pull images from VM 1's registry. **Important:** Replace `<VM1_PUBLIC_IP>` with VM 1's public IP address.
```bash
echo '{"insecure-registries": ["<VM1_PUBLIC_IP>:80"]}' | sudo tee /etc/docker/daemon.json
sudo systemctl restart docker
```

### Step 5: Authorize Jenkins SSH Access
On **VM 1**, get the public key:
```bash
cat ~/.ssh/id_rsa.pub
```
Copy the entire output.

On **VM 2**, add the key to `authorized_keys`:
```bash
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
```
Paste VM 1's public key, then save and exit. Secure the permissions:
```bash
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```
Test the connection from **VM 1**: `ssh ubuntu@<VM2_PUBLIC_IP>`. You should connect without a password.

### Step 6: Prepare Application Directory
On **VM 2**, create the application directory:
```bash
mkdir ~/devspace && cd ~/devspace
nano docker-compose.yml
```
Paste this configuration, replacing placeholders with your values:
```yaml
services:
  portfolio:
    image: <VM1_PUBLIC_IP>:80/<FORGEJO_USERNAME>/<REPO_NAME>:latest
    container_name: portfolio-app
    ports:
      - "80:80"
    restart: unless-stopped
```
Save and exit.

---

## Goal: Configure your development environment and connect it to Forgejo.

### Step 1: Configure Git Identity
In your local terminal, run:
```bash
git config --global user.name "Developer"
git config --global user.email "dev@example.com"
```

### Step 2: Generate and Add SSH Key to Forgejo
1.  Generate a new SSH key: `ssh-keygen -t rsa -b 4096` (accept defaults).
2.  Display the public key: `cat ~/.ssh/id_rsa.pub`.
3.  Copy the key.
4.  In the Forgejo UI, go to your **Profile Picture** -> **Settings** -> **SSH / GPG Keys**.
5.  Click **Add Key**, give it a name, paste the public key content, and click **Add Key**.

### Step 4: Initialize Git Repository
In your local project directory:
```bash
# Navigate to or create your project directory
cd ~/devspace

# Initialize Git
git init

# Add Forgejo as the remote origin
git remote add origin ssh://git@<VM1_PUBLIC_IP>:2222/<YOUR_USERNAME>/<YOUR_REPO>.git
```
**Note:** Use port `2222` for SSH with Forgejo.

### Step 5: Test SSH Connection
```bash
ssh -T git@<VM1_PUBLIC_IP> -p 2222
```
Type `yes` if prompted. A success message confirms your connection.

---

## Goal: Create the Jenkinsfile, commit your code, and trigger the automated CI/CD pipeline.

### Step 1: Create the Jenkinsfile
In your local project directory, create a file named `Jenkinsfile` and add the following content.
```groovy
pipeline {
    agent any

    environment {
        REGISTRY = "<VM1_PUBLIC_IP>:80"              // VM 1's public IP with port 80
        IMAGE_NAME = "<USERNAME>/<REPO_NAME>"        // Your Forgejo username and repo name
        VM2_IP = "<VM2_PUBLIC_IP>"                   // VM 2's public IP
        VM2_SSH_CRED_ID = "vm2-ssh"                  // Jenkins credential ID for SSH
        FORGEJO_CRED_ID = "forgejo-registry"         // Jenkins credential ID for registry
    }
    
    // Add stages for build, push, and deploy here
}
```
Update the placeholder values in the `environment` block.

### Step 2: (Optional) Update Profile Picture and Config
You can update your application files, such as `config.js` or add a profile picture. Example:
```bash
# Download a profile picture
curl -s https://api.github.com/users/<YOUR_GITHUB_USERNAME> | grep -o 'https://avatars.githubusercontent.com/u/[^"]*' | xargs curl -o profile_picture.jpg
```

### Step 4: Commit and Push
Add, commit, and push your files to Forgejo. This will trigger the pipeline.
```bash
git add .
git commit -m "feat: complete end-to-end CI/CD setup"
git push -u origin master
```

### Step 5: Monitor the Pipeline
Go to your Jenkins dashboard at `http://<VM1_PUBLIC_IP>/jenkins`. You should see your pipeline running. Click on the build and view the **Console Output** to monitor the stages: Build, Push, and Deploy.

### Step 6: Verify Deployment
Once the pipeline succeeds, navigate to `http://<VM2_PUBLIC_IP>/` in your browser to see your deployed application. Any new pushes to the repository will automatically trigger a new deployment.

---

## What We've Built
*   **Self-Hosted Git Server:** Forgejo running on VM 1 with webhook integration.
*   **Automation Server:** Jenkins configured to listen for Git events and orchestrate deployments.
*   **Container Registry:** Docker registry hosted on VM 1 for storing application images.
*   **Production Environment:** VM 2 running your containerized application.
*   **Automated Pipeline:** A complete `code push -> Build -> Test -> Deploy` workflow.

### Key Concepts Implemented
*   **Webhook Integration:** Automatic pipeline triggering on code push.
*   **Containerization:** Docker for consistent application packaging.
*   **Registry Management:** Private Docker registry for image storage.
*   **SSH Automation:** Passwordless deployment to production servers.
*   **Reverse Proxy:** Nginx routing traffic to multiple services.
*   **Infrastructure as Code:** Docker Compose for reproducible deployments.
