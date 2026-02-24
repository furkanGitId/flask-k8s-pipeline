# CI/CD Pipeline Setup Guide

## Component Overview

| Component | Purpose           | Port |
|-----------|-------------------|------|
| Docker    | Container runtime | —    |
| Jenkins   | CI/CD automation  | 8080 |
| SonarQube | Code quality      | 9000 |
| MSSQL     | Database          | 1433 |
| Node API  | Backend           | 5000 |
| React     | Frontend          | 3000 |

---

## 1. Docker Installation

```bash
sudo apt update
sudo apt install docker.io docker-compose -y
sudo usermod -aG docker $USER && newgrp docker
systemctl status docker
docker --version
docker-compose version
docker ps -a
docker images -a
docker run hello-world
docker run -d -p 8081:80 --name my-web-server nginx
docker stop my-web-server
docker rm my-web-server
```

---

## 2. Installation of Java

```bash
sudo apt update
sudo apt install fontconfig openjdk-21-jre
java -version
```

### (a) Install Jenkins

```bash
# Download the Jenkins GPG key
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

**Install additional plugins:**

Go to: `Manage Jenkins → Plugins → Available plugins` → search and install:

- Docker Pipeline
- Docker Commons Plugin
- GitHub Integration Plugin
- SonarQube Scanner
- NodeJS Plugin

---

## 3. Setup SonarQube

> We will run SonarQube as a Docker container for simplicity.

```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:lts-community
```

Access at: `http://YOUR_SERVER_IP:9000`

> **Default login:** `admin / admin` (you'll be prompted to change it)

### (a) Generate a Token for Jenkins

Navigate to: `SonarQube → My Account → Security → Generate Token`

- **Name:** `jenkins-token`
- **Type:** `Global Analysis Token`
- Copy the token (shown only once!)

```
sqa_dbdec3e7924381623b9a6f516eef3d53c062582e
```

### (b) Add Token to Jenkins Credentials

Navigate to: `Manage Jenkins → Credentials → System → Global → Add Credentials`

- **Kind:** Secret text
- **Secret:** (paste token)
- **ID:** `sonarqube-token`

### (c) Configure SonarQube Server in Jenkins

Navigate to: `Manage Jenkins → System → SonarQube servers → Add`

- **Name:** `SonarQube`
- **URL:** `http://YOUR_IP:9000`
- **Token:** select `sonarqube-token`

---

## 4. Install ngrok

```bash
curl -s https://ngrok-agent.s3.amazonaws.com/ngrok.asc | \
  sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null

echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | \
  sudo tee /etc/apt/sources.list.d/ngrok.list

sudo apt update && sudo apt install -y ngrok
```

### (a) Authenticate

Sign up at [https://ngrok.com](https://ngrok.com) → Dashboard → copy your authtoken

- Setup guide: https://dashboard.ngrok.com/get-started/setup/windows
- Auth token page: https://dashboard.ngrok.com/get-started/your-authtoken

```bash
ngrok config add-authtoken 39psetxv4SMe5nUEtew3lE55DqX_6Fg76AY3LDscq7x8BRRU9
ngrok http 8080
```

**Forwarding:**
```
https://shaneka-plummier-untreacherously.ngrok-free.dev -> http://localhost:8080
```

---

## 5. GitHub Repository Setup

> Suggested GitHub repo name: `flask-k8s-pipeline`

```bash
echo "# todo-cicd-pipeline" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/furkanGitId/flask-k8s-pipeline.git
git push -u origin main
```

### (a) Set up GitHub Webhook

Navigate to: `GitHub repo → Settings → Webhooks → Add webhook`

- **Payload URL:** `https://YOUR-NGROK-URL/github-webhook/`
- **Content type:** `application/json`
- **Events:** Push + Pull requests
- **Active:** ✓ → Save

---

## 6. Create the Pipeline Job

Navigate to: `Jenkins → New Item → Name: flask-k8s-pipeline → Pipeline → OK`

- **General:** check `GitHub project` → paste repo URL
- **Build Triggers:** check `GitHub hook trigger for GITScm polling`
- **Pipeline:** Definition = `Pipeline script from SCM`
- **SCM:** Git → Repository URL: your GitHub URL
- **Branch:** `*/main`
- **Script Path:** `Jenkinsfile`
- Click **Save**

---

## 7. Configure SonarQube Scanner

Navigate to: `Manage Jenkins → Global Tool Configuration → SonarQube Scanner`

---

## 8. Fix Node.js Version (if not installed)

> If node version is not installed, run the below commands:

```bash
# Remove the incomplete Node.js 18 install
sudo apt-get remove -y nodejs

# Install Node.js 22 properly via NodeSource (includes npm)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs

# Verify both as the jenkins user
sudo -u jenkins node -v    # should print v22.x.x
sudo -u jenkins npm -v     # should now work
```

---

## 9. Grant Permissions to Jenkins

```bash
echo "jenkins ALL=(ALL) NOPASSWD: /usr/bin/apt-get" | sudo tee /etc/sudoers.d/jenkins
```

---

## 10. Fix SonarQube Timeout Error

If you encounter the following error:

```
Checking status of SonarQube task 'AZx61Sfz9xLaYDsVjPAX' on server 'sonarqube'
SonarQube task 'AZx61Sfz9xLaYDsVjPAX' status is 'IN_PROGRESS'
Cancelling nested steps due to timeout
[Pipeline] }
```

**Solution:** Go to `SonarQube → Administration → Configuration → Webhooks → Create` and set the webhook URL.

First, find your machine's IP:

```bash
hostname -I
```

**Example output:**
```
furkan@DESKTOP-M5SA76T:~$ hostname -I
172.31.66.217 172.17.0.1 172.18.0.1 172.19.0.1
```

> It will print something like `172.x.x.x` or `192.168.x.x`. Use that IP in the webhook URL.

**Then set the webhook URL as:**

```
http://172.31.66.217:8080/sonarqube-webhook/
http://172.x.x.x:8080/sonarqube-webhook/
```

---

## 10. GitHub Personal Access Token (PAT)

### Option 1: Personal Access Token (PAT)

> This is the most common method. It acts as a "password" for your specific GitHub account.

#### Step A: Generate the Token on GitHub

1. Go to your GitHub **Settings** (click your profile icon → Settings).
2. Scroll down to **Developer settings → Personal access tokens → Tokens (classic)**.
3. Click **Generate new token (classic)**.
4. Give it a name (e.g., `"Jenkins-Webhook-Token"`).
5. **Scopes to select:**
   - `repo` (Full control of private repositories)
   - `admin:repo_hook` (Full control of repository hooks — This is vital for fixing the 403 error)
6. Click **Generate token** and copy it immediately. You won't see it again.

#### Step B: Add to Jenkins

1. Go to `Manage Jenkins → Credentials`.
2. Click `(global) → Add Credentials`.
3. **Kind:** Select `Secret text`.
4. **Secret:** Paste the GitHub token here.
5. **ID:** Give it a name like `github-token`.
6. Go to `Manage Jenkins → System → GitHub section`.
7. Click **Add GitHub Server** → Select your new credential in the dropdown → Click **Test Connection**.

```
ghp_secret
```

---

## 11. Install kubectl and Minikube

```bash
# Install kubectl first
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
    https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify kubectl
kubectl version --client

# Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start Minikube using Docker driver
minikube start --driver=docker

# Verify Minikube is running
minikube status
kubectl get nodes
```

**Expected output:**

```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1m    v1.28.x
```

### (a) Fix Permission / Error — Clean Everything and Restart Properly (Most Reliable)

> As root (or your `furkan` user where it might work):

```bash
minikube delete --all --purge   # or sudo minikube delete --all
rm -rf /var/lib/jenkins/.minikube /var/lib/jenkins/.kube
```

**Otherwise start Minikube:**

```bash
# Then as jenkins:
sudo -u jenkins minikube start --driver=docker
```

**Verify status:**

```bash
sudo -u jenkins minikube status
sudo -u jenkins kubectl get nodes
sudo -u jenkins bash -c 'eval $(minikube docker-env) && docker ps'
```

**Port forwarding:**

```bash
furkan@DESKTOP-M5SA76T:~$ sudo KUBECONFIG=/var/lib/jenkins/.kube/config kubectl port-forward service/my-app-service 30080:80
Forwarding from 127.0.0.1:30080 -> 5000
Forwarding from [::1]:30080 -> 5000
Handling connection for 30080
Handling connection for 30080
```

**Access the app at:** `http://localhost:30080/`
