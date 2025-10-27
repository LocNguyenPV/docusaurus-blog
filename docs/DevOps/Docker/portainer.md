---
sidebar_position: 6
---

# Multi-host deployment and management using Portainer
## TL;DR
I will simulate a multi-host environment using DinD and then use Portainer for deployment and management.

## Prerequisite
- Installed Docker (If you don't have it, follow the instructions for [Windows](https://dev.to/locnguyenpv/install-docker-on-window-5cg0) or [Linux](https://dev.to/locnguyenpv/install-docker-on-linux-2k1k))

- Read [previous post](https://dev.to/locnguyenpv/deploy-microservice-with-docker-from-scratch-p1-46h7) to know what I done

## Architecture
I will migrate from a single-host to a multi-host architecture.

![Architecture](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/surc46qa14nqxybu23p4.gif)
<figCaption>Multi-host</figCaption>

## Install Portainer
**Step 1:** Register 3 nodes free for portainer
- Click on the [link](https://www.portainer.io/take-3)
- Fill out the form
- Get an email and save the license key

![License Key](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bv2ol8mp20vxkykktind.png)
<figCaption>Email license key</figCaption>

**Step 2:** Create docker volume for portainer
```batch
docker volume create portainer_data
```

**Step 3:** Create network
```batch
docker network create portainer-network
```

**Step 4:** Run portainer with specify network

```batch
docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always --network portainer-network -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ee:lts  
```

**Step 5:** Access portainer via <u>localhost:9443/#!/endpoints</u> to create account

![register account](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7yxetbl08cc31lnoglp2.png)
<figCaption>Register Account</figCaption>

**Step 6:** Register license key you get from email 

![register license key](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gp7lur6pfop6ds67g7a3.png)
<figCaption>Register key</figCaption>


![home page](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ix2uqwm1vrecdvqigqi2.png)
<figCaption>Home page</figCaption>

## Push image to docker registry
**Step 1:** Open terminal and run (it will will be prompted to enter Docker ID (username) and password interactively)

```batch
docker login
```
**Step 2:** Tagging image with the following format `<your-username-docker>/<application-name>:<version>`. Your username should be the same display on docker

![docker hub](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gpzzb2bifi1x0w5cgljl.png)

For me, username is `locnguyenpv`

- FE

```batch
docker tag project-management/client-app locnguyenpv/client-app:v1
```

- BE

```batch
docker tag project-management/project-service locnguyenpv/project-service:v1
docker tag project-management/user-service locnguyenpv/user-service:v1
docker tag project-management/task-service locnguyenpv/task-service:v1
```

**Step 3:** Push image to registry by tag
- FE

```batch
docker push locnguyenpv/client-app:v1
```

- BE

```batch
docker push locnguyenpv/project-service:v1
docker push locnguyenpv/user-service:v1
docker push locnguyenpv/task-service:v1
```

## Connect to DinD
**Step 1:** Create 2 DinD, one for FE, one for BE
- FE:
```batch
docker run --privileged -d --name project-management-fe --network portainer-network -p 80:80 -p 9002:9001 -p 2377:2375 docker:dind 
```
- BE: 

```batch
docker run --privileged -d --name project-management-be --network portainer-network -p 2376:2375 -p 9001:9001 -p 9099:9099 docker:dind 
```

**Step 2:** SSH into each DinD to install Portainer Agent
- FE:
```batch
docker exec -it project-management-fe sh
docker run -d \
  -p 9001:9001 \
  --name portainer_agent \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -v /:/host \
  portainer/agent:2.27.9
```
- BE:
```batch
docker exec -it project-management-be sh
docker run -d \
  -p 9001:9001 \
  --name portainer_agent \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -v /:/host \
  portainer/agent:2.27.9
```

**Step 3:** Connect DinD with Portainer
![Environment Dashboard](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/q2x7r4do238hvugkywa3.png)

![select environment](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/q6o388r7l9xj0frbd3q5.png)

![connect env](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jqzx65967qz0agtk6ozk.png)


![connect success](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/g30zqv0xnlmenpfam927.png)
<figCaption>Connect Success</figCaption>

## Update docker-compose
Now that I've pushed the image to the Docker registry, I can use it instead of building from source. <u>Replace build from source to pull image from registry</u>
- FE: 

```yaml
services:
  client-service:
    image: locnguyenpv/client-app:v1
    ports:
      - "80:80"
    restart: always
```

- BE: 

```yaml
services:
  mysql-service:
    image: mysql:latest
    container_name: mysql-container
    environment:
      MYSQL_PASSWORD: 123123
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql  # create space for persistence data
      - ./database:/docker-entrypoint-initdb.d # Mount init file into container 
    healthcheck:  # Check every 10s to make sure it work fine
      test: ["CMD-SHELL", "mysqladmin -u root -p$MYSQL_PASSWORD ping -h localhost || exit 1"]
      interval: 30s
      retries: 5
      timeout: 10s
      start_period: 10s
    networks:
      - internal-network

  nginx:
    image: nginx:latest
    container_name: nginx-gateway
    ports:
      - "9099:9099"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro # Mount nginx config from local to container
    depends_on:        # Start condition
      - project-service
      - user-service
      - task-service
    networks:
      - internal-network   # Virtual network
  project-service:
    image: locnguyenpv/project-service:v1
    container_name: project-service-container
    ports:
      - "8081:8081"
    environment:
      MYSQL_HOST: mysql-service
      MYSQL_PORT: 3306
      MYSQL_USER: root
      MYSQL_PASSWORD: 123123
    depends_on:
      mysql-service:
        condition: service_healthy
    networks:
      - internal-network # Virtual network
  user-service:
    image: locnguyenpv/user-service:v1
    container_name: user-service-container
    ports:
      - "8082:8082"
    environment:
      MYSQL_HOST: mysql-service
      MYSQL_PORT: 3306
      MYSQL_USER: root
      MYSQL_PASSWORD: 123123
    depends_on:
      mysql-service:
        condition: service_healthy
    networks:
      - internal-network # Virtual network
  task-service:
    image: locnguyenpv/task-service:v1
    container_name: task-service-container
    ports:
      - "8083:8083"
    environment:
      MYSQL_HOST: mysql-service
      MYSQL_PORT: 3306
      MYSQL_USER: root
      MYSQL_PASSWORD: 123123
    depends_on:
      mysql-service:
        condition: service_healthy
    networks:
      - internal-network # Virtual network

networks:
  internal-network:
      driver: bridge
      name: internal-network

volumes:
  mysql-data: 
```

## Deploy to DinD

**Step 1:** Go to home page and choose DinD 

![home page](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/yjxdexh4z0ww0jakrjgu.png)

**Step 2:** Choose stack 

![stack](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/g2oefqc2bnwoxagwu8wa.png)

**Step 3:** Add new stack

![add stack](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tpkzga6j9kustn4p69o4.png)

**Step 4:** Copy & paste docker-compose file into editor

![stack editor](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/dpvqk3vae6krz99hz7e5.png)

**Step 5:** Deploy

![deploy stack](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/l6o48g1bt7fpspy6goar.png)



![deploy success](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o65xcw91ctxl1ijf876d.png)
<figCaption>Deploy success</figCaption>

Replace those steps for other DinD. All set!

## Conclusion
I just show you the way how to simulate multi-host environment by DinD. After that, use Portainer for management and deployment.

Happy Coding 

