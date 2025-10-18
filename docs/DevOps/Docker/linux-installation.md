---
sidebar_position: 2
---

# Install Docker on Linux
## 1) Update system
- Make sure your system are using latest version of everything

```bash
sudo apt update
sudo apt upgrade -y
```
- Install packages help `apt ` download package via HTTPS

```bash
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
```

## 2) Install Docker

Docker can be installed using either a quick method or a more detailed one.

### The quick one

```bash
curl -fsSL https://get.docker.com/ | sh
```

### Another way
- Add Docker's GPG official

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg |sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

- Add Docker repository into APT source list

```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list> /dev/null
```

- Install Docker Engine

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y
```

## 3) Check Docker status
If it's fine, you'll see `active (running)` status

```bash
sudo systemctl status docker
```

## 4) Allow Docker run without `sudo`

```bash
sudo usermod -aG docker ${USER}
newgrp docker
```

## 5) Test Docker
If you see welcome message, it mean Docker install successfully 

```bash
docker run hello-world

```

## 6) Install Docker Compose (Optional)

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose sudo chmod +x /usr/local/bin/docker-compose
```
Now your Linux system ready to go!
Happy Coding.  
