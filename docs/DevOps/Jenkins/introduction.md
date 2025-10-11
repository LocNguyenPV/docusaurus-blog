# Introduction 

1) Cài đặt jenkin với docker
3) Truy cập Jenkin UI
4) Lấy init password
5) Setup Jenkin
6) Install plugin Role-based Authorization Strategy
7) Install gitlab plugin
8) Setting Authorization với role-based trong Security  
9) Cấu hình gitlab trong setting
10) Cấu hình role cho user

```dockerfile title="/jenkins/Dockerfile"
FROM jenkins/jenkins:lts
USER root
 
# install curl, kubectl and docker CLI
RUN apt-get update \
  && apt-get install -y ca-certificates curl apt-transport-https gnupg2 lsb-release docker.io \
  && curl -fsSL "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" -o /usr/local/bin/kubectl \
  && chmod +x /usr/local/bin/kubectl /usr/bin/docker || true \
  && apt-get clean && rm -rf /var/lib/apt/lists/*
 
USER jenkins
```

```yaml title="/compose.yaml"
services:
  jenkins:
    build:
      context: ./jenkins
    container_name: jenkins
    restart: unless-stopped
    privileged: true
    user: root
    ports:
      - "3000:8080"   # Jenkins UI
      - "50000:50000" # Agent communication
    volumes:
      #- /home/nhanjs/.kube:/root/.kube           # mount local kube config
      #- /home/nhanjs/.minikube:/root/.minikube   # mount minikube certs and keys
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock # Allows Jenkins to run docker commands
    networks:
      - gitlab-net
 
  jenkins-agent:
    image: jenkins
    container_name: jenkins-agent
    restart: unless-stopped
    privileged: true
    extra_hosts:
      - "jenkins.local:host-gateway"
    environment:
      - JENKINS_URL=http://jenkins:8080
      - JENKINS_AGENT_NAME=docker-agent
      - JENKINS_SECRET=<SECRET>
      - JENKINS_WEB_SOCKET=true
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /home/jenkins/agent:/home/jenkins/agent
    networks:
      - gitlab-net
volumes:
  jenkins_home:
    name: jenkins_home
networks:
  gitlab-net:
    name: gitlab-net
```
```bash title="get Jenkins init password"
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

```yaml title="pipeline"
pipeline {
    agent { label 'docker-agent' }
 
    environment {
        // GitLab Container Registry host and project (namespace/project)
        GITLAB_REGISTRY       = "gitlab.local:5050"
        GITLAB_PROJECT        = "devops/core2"
        GITLAB_CREDENTIALS    = "gitlab-cr"
 
        FRONTEND_IMAGE = "${env.GITLAB_REGISTRY}/${env.GITLAB_PROJECT}/core2-frontend:latest"
        BACKEND_IMAGE  = "${env.GITLAB_REGISTRY}/${env.GITLAB_PROJECT}/core2-backend:latest"
    }
 
    stages {
        stage('Check Docker') {
            steps {
                sh 'docker --version'
            }
        }
 
        stage('Build Frontend') {
            steps {
                dir('frontend') {
                    sh """
                        docker build -t ${FRONTEND_IMAGE} -f Dockerfile .
                    """
                }
            }
        }
 
        stage('Build Backend') {
            steps {
                dir('CoreAPI') {
                    sh """
                        docker build -t ${BACKEND_IMAGE} -f Dockerfile .
                    """
                }
            }
        }
 
        stage('Push Images') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: env.GITLAB_CREDENTIALS, usernameVariable: 'GITLAB_USER', passwordVariable: 'GITLAB_PASS')]) {
                        sh """
                            echo "$GITLAB_PASS" | docker login ${GITLAB_REGISTRY} -u "$GITLAB_USER" --password-stdin
                            docker push ${FRONTEND_IMAGE}
                            docker push ${BACKEND_IMAGE}
                            docker logout ${GITLAB_REGISTRY}
                        """
                    }
                }
            }
        }
 
        stage('Confirm from CTO') {
            steps {
                script {
                    timeout(time: 1, unit: 'HOURS') {
                        input message: "CTO approval required to push Docker images. Proceed?",
                        ok: "Approve and Continue",
                        submitter: "CTO"
                    }
                }
            }
        }
 
        stage('Deploy to K8s (Minikube)') {
            steps {
                script {
                    echo "Deploying to local Minikube..."
                    sh '''
                        kubectl version --client
                        kubectl config use-context minikube
                        kubectl apply -f k8s/backend-deployment.yaml
                        kubectl apply -f k8s/backend-service.yaml
                        kubectl apply -f k8s/frontend-deployment.yaml
                        kubectl apply -f k8s/frontend-service.yaml
                    '''
 
                    echo "Deployment complete. Access frontend via:"
                    sh 'minikube service core2-frontend --url'
                }
            }
        }
    }
}
```