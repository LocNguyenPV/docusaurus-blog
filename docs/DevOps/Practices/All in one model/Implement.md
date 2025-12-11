# Implement

**Key Components:**

- **GCP Compute Engine VM**: Single VM hosting all services
- **Docker**: Runs all services as containers
- **Nginx**: Reverse proxy to route traffic
- **Jenkins**: CI/CD automation
- **GitLab**: Self-hosted Git with built-in CI
- **GitLab Registry**: Private Docker image registry
- **Firewall**: GCP firewall rules to control access

---

## Prerequisites

Before starting, you need:

- **GCP Account** with billing enabled
- **gcloud CLI** installed and authenticated
- **Basic knowledge** of Docker, Linux, and Git
- **Domain name** (optional, for production SSL setup)

---

## Step 1: Create a GCP Compute Engine VM

### 1.1 Create the VM via gcloud CLI

```bash
gcloud compute instances create devops-vm \
  --zone=us-central1-a \
  --machine-type=e2-standard-4 \
  --boot-disk-size=50GB \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --tags=jenkins,gitlab,nginx,ssh-server
```

**Explanation:**

- `--machine-type=e2-standard-4`: 4 vCPUs, 16GB RAM (minimum for Jenkins + GitLab)
- `--boot-disk-size=50GB`: Storage for Docker images and GitLab data
- `--tags`: Network tags for firewall rules (we'll use these later)

### 1.2 SSH into the VM

```bash
gcloud compute ssh devops-vm --zone=us-central1-a
```

Once connected, you should see the Ubuntu terminal.

---

## Step 2: Install Docker and Docker Compose

### 2.1 Install Docker

```bash
# Update packages
sudo apt-get update

# Install dependencies
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common

# Add Docker GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# Add your user to docker group
sudo usermod -aG docker $USER

# Apply group changes
newgrp docker
```

**Verify Docker installation:**

```bash
docker --version
```

### 2.2 Install Docker Compose

```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

docker-compose --version
```

---

## Step 3: Create Docker Compose Configuration

Create a directory for your setup:

```bash
mkdir ~/devops-stack
cd ~/devops-stack
```

Create a `docker-compose.yml` file:

```yaml
version: "3.8"

services:
  # Nginx Reverse Proxy
  nginx:
    image: nginx:latest
    container_name: nginx-proxy
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - jenkins
      - gitlab
    restart: always
    networks:
      - devops-network

  # Jenkins
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_data:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - JENKINS_OPTS="--prefix=/jenkins"
    user: root
    restart: always
    networks:
      - devops-network

  # GitLab
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    hostname: gitlab.example.com
    ports:
      - "8081:80"
      - "443:443"
      - "22:22"
      - "5380:5380"
    volumes:
      - gitlab_config:/etc/gitlab
      - gitlab_logs:/var/log/gitlab
      - gitlab_data:/var/opt/gitlab
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.example.com:8081'
        gitlab_rails['gitlab_shell_ssh_port'] = 22
        registry_external_url 'http://gitlab.example.com:5380'
    restart: always
    networks:
      - devops-network

volumes:
  jenkins_data:
  gitlab_config:
  gitlab_logs:
  gitlab_data:

networks:
  devops-network:
    driver: bridge
```

---

## Step 4: Configure Nginx Reverse Proxy

Create the Nginx configuration directory and file:

```bash
mkdir -p ~/devops-stack/nginx
```

Create `~/devops-stack/nginx/nginx.conf`:

```nginx
events {
    worker_connections 1024;
}

http {
    upstream jenkins {
        server jenkins:8080;
    }

    upstream gitlab {
        server gitlab:80;
    }

    upstream gitlab-registry {
        server gitlab:5380;
    }

    server {
        listen 80;
        server_name jenkins.example.com;

        location / {
            proxy_pass http://jenkins;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    server {
        listen 80;
        server_name gitlab.example.com;

        location / {
            proxy_pass http://gitlab;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    server {
        listen 80;
        server_name registry.example.com;

        location / {
            proxy_pass http://gitlab-registry;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

**Note:** Replace `example.com` with your actual domain, or use the VM's external IP for testing.

---

## Step 5: Configure GCP Firewall Rules

Open the necessary ports on GCP firewall for HTTP, Jenkins, GitLab, and SSH.

### Via gcloud CLI:

```bash
# Allow HTTP traffic (port 80)
gcloud compute firewall-rules create allow-http \
  --direction=INGRESS \
  --priority=1000 \
  --network=default \
  --action=ALLOW \
  --rules=tcp:80 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=nginx

# Allow Jenkins (port 8080)
gcloud compute firewall-rules create allow-jenkins \
  --direction=INGRESS \
  --priority=1000 \
  --network=default \
  --action=ALLOW \
  --rules=tcp:8080 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=jenkins

# Allow GitLab UI (port 8081)
gcloud compute firewall-rules create allow-gitlab \
  --direction=INGRESS \
  --priority=1000 \
  --network=default \
  --action=ALLOW \
  --rules=tcp:8081 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=gitlab

# Allow GitLab Registry (port 5380)
gcloud compute firewall-rules create allow-gitlab-registry \
  --direction=INGRESS \
  --priority=1000 \
  --network=default \
  --action=ALLOW \
  --rules=tcp:5380 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=gitlab

# Allow SSH (port 22)
gcloud compute firewall-rules create allow-ssh \
  --direction=INGRESS \
  --priority=1000 \
  --network=default \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=ssh-server
```

### Via GCP Console:

1. Go to **VPC Network → Firewall**
2. Click **Create Firewall Rule**
3. For each rule:
   - **Name:** `allow-http`, `allow-jenkins`, etc.
   - **Direction:** Ingress
   - **Targets:** Specified target tags (e.g., `nginx`, `jenkins`, `gitlab`)
   - **Source IP ranges:** `0.0.0.0/0` (all IPs)
   - **Protocols and ports:** `tcp:80`, `tcp:8080`, `tcp:8081`, `tcp:5380`, `tcp:22`
4. Click **Create**

---

## Step 6: Start All Services

From the `~/devops-stack` directory:

```bash
docker-compose up -d
```

This will start:

- Nginx on port 80
- Jenkins on port 8080
- GitLab on ports 8081 (UI), 22 (SSH), 5380 (Registry)

**Check running containers:**

```bash
docker ps
```

You should see `nginx-proxy`, `jenkins`, and `gitlab` running.

---

## Step 7: Access and Configure Jenkins

### 7.1 Get Jenkins Initial Admin Password

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Copy the password.

### 7.2 Access Jenkins UI

Open your browser and go to:

```
http://<VM_EXTERNAL_IP>:8080
```

Or if using Nginx proxy with domain:

```
http://jenkins.example.com
```

**Login:**

- Paste the initial admin password
- Install suggested plugins
- Create your first admin user

### 7.3 Install Docker Plugin in Jenkins

1. Go to **Manage Jenkins → Manage Plugins**
2. Search for **Docker** and **Docker Pipeline**
3. Install and restart Jenkins

---

## Step 8: Access and Configure GitLab

### 8.1 Get GitLab Root Password

GitLab generates a random password on first startup. Wait 3-5 minutes for GitLab to fully boot, then:

```bash
docker exec gitlab grep 'Password:' /etc/gitlab/initial_root_password
```

Copy the password.

### 8.2 Access GitLab UI

```
http://<VM_EXTERNAL_IP>:8081
```

Or with domain:

```
http://gitlab.example.com:8081
```

**Login:**

- Username: `root`
- Password: (from previous step)

### 8.3 Create a GitLab Project

1. Click **New project**
2. Name it `demo-app`
3. Set visibility to **Private**
4. Click **Create project**

---

## Step 9: Configure GitLab Registry

GitLab's built-in Docker registry is now accessible at port 5380.

### 9.1 Login to GitLab Registry from VM

```bash
docker login <VM_EXTERNAL_IP>:5380
# Username: root
# Password: (your GitLab root password)
```

### 9.2 Push a Test Image

```bash
# Tag an image
docker tag nginx:latest <VM_EXTERNAL_IP>:5380/demo-app/nginx:v1.0

# Push to GitLab registry
docker push <VM_EXTERNAL_IP>:5380/demo-app/nginx:v1.0
```

Check in GitLab UI: **Project → Packages & Registries → Container Registry**.

---

## Step 10: Integrate Jenkins with GitLab

### 10.1 Create GitLab Personal Access Token

1. In GitLab, go to **User Settings → Access Tokens**
2. Create token with scopes: `api`, `read_repository`, `write_repository`
3. Copy the token

### 10.2 Add GitLab Credentials in Jenkins

1. Go to **Manage Jenkins → Manage Credentials**
2. Click **Add Credentials**
3. Kind: **GitLab API token**
4. Paste the token
5. ID: `gitlab-token`

### 10.3 Install GitLab Plugin in Jenkins

1. Go to **Manage Jenkins → Manage Plugins**
2. Search for **GitLab**
3. Install and restart

### 10.4 Configure GitLab in Jenkins

1. Go to **Manage Jenkins → Configure System**
2. Find **GitLab** section
3. Add GitLab connection:
   - **Connection name:** `gitlab-local`
   - **GitLab host URL:** `http://gitlab:80` (Docker internal network)
   - **Credentials:** Select `gitlab-token`
4. Click **Test Connection** → should show success
5. Save

---

## Step 11: Create a Jenkins Pipeline

### 11.1 Create Pipeline Job

1. In Jenkins, click **New Item**
2. Name: `demo-app-pipeline`
3. Type: **Pipeline**
4. Click **OK**

### 11.2 Configure Pipeline

In the pipeline configuration:

**Build Triggers:**

- Check **Build when a change is pushed to GitLab**

**Pipeline Script:**

```groovy
pipeline {
    agent any

    environment {
        GITLAB_REGISTRY = '<VM_EXTERNAL_IP>:5380'
        PROJECT_NAME = 'demo-app'
        IMAGE_NAME = "${GITLAB_REGISTRY}/${PROJECT_NAME}/app"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git credentialsId: 'gitlab-token',
                    url: 'http://gitlab/root/demo-app.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Push to GitLab Registry') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'gitlab-credentials',
                        usernameVariable: 'USER',
                        passwordVariable: 'PASS'
                    )]) {
                        sh "echo $PASS | docker login -u $USER --password-stdin ${GITLAB_REGISTRY}"
                        sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}
```

Save the pipeline.

---

## Step 12: Set Up GitLab Webhook to Trigger Jenkins

### 12.1 Get Jenkins Webhook URL

In your Jenkins pipeline job, find the GitLab webhook URL:

```
http://<VM_EXTERNAL_IP>:8080/project/demo-app-pipeline
```

### 12.2 Add Webhook in GitLab

1. In GitLab project, go to **Settings → Webhooks**
2. **URL:** `http://jenkins:8080/project/demo-app-pipeline`
3. **Secret Token:** (leave blank or generate one)
4. **Trigger:** Check **Push events**
5. Uncheck **Enable SSL verification** (for testing)
6. Click **Add webhook**
7. Click **Test → Push events**

You should see a successful response, and Jenkins should trigger a build.

---

## Step 13: Test the Complete Pipeline

### 13.1 Clone GitLab Repo Locally

```bash
git clone http://<VM_EXTERNAL_IP>:8081/root/demo-app.git
cd demo-app
```

### 13.2 Create a Simple Dockerfile

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
```

Create `index.html`:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Demo App</title>
  </head>
  <body>
    <h1>Hello from CI/CD Pipeline!</h1>
    <p>Built by Jenkins, stored in GitLab Registry</p>
  </body>
</html>
```

### 13.3 Push Changes

```bash
git add .
git commit -m "Add Dockerfile and index.html"
git push origin main
```

### 13.4 Watch Jenkins Build

- GitLab webhook triggers Jenkins
- Jenkins clones code
- Jenkins builds Docker image
- Jenkins pushes to GitLab Registry
- Check Jenkins console output for success

---

## Security Best Practices

1. **Change default passwords immediately** (Jenkins admin, GitLab root)
2. **Restrict firewall source IPs** to your office/VPN instead of `0.0.0.0/0`
3. **Enable HTTPS** using Let's Encrypt (Certbot + Nginx)
4. **Use SSH keys** for GitLab access instead of passwords
5. **Enable 2FA** on GitLab and Jenkins
6. **Regular backups** of Docker volumes (`jenkins_data`, `gitlab_data`)
7. **Limit Docker socket access** (Jenkins running as root is convenient but risky in production)

---

## Troubleshooting

**Jenkins can't access GitLab:**

- Use Docker network names: `http://gitlab:80` not `http://localhost:8081`
- Check `docker network inspect devops-stack_devops-network`

**GitLab Registry push fails:**

- Ensure you're logged in: `docker login <VM_IP>:5380`
- Check GitLab logs: `docker logs gitlab`

**Firewall blocking ports:**

- Verify rules: `gcloud compute firewall-rules list`
- Test connectivity: `telnet <VM_IP> 8080`

**Containers keep restarting:**

- Check logs: `docker logs <container_name>`
- Increase VM resources (Jenkins + GitLab need 16GB+ RAM)

---

## Conclusion

You now have a fully functional CI/CD infrastructure running on a single GCP VM with:

- Jenkins for pipeline automation
- GitLab for Git hosting and container registry
- Nginx as a reverse proxy
- Docker for containerization
- GCP firewall for security

This setup is perfect for learning, small teams, or development environments. For production, consider splitting services across multiple VMs or using managed services like Cloud Build, Artifact Registry, and GKE.
