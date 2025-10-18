---
sidebar_position: 4
---

# Deploy microservice with Docker from scratch - P2
## TL;DR
Use Docker Compose to run and deploy a project's Dockerfiles as microservices.

## Docker-compose
In the [previous post](https://dev.to/locnguyenpv/deploy-microservice-with-docker-from-scratch-p1-46h7), I have an overview of the project and its functionality. In this post, I'll delve into implementation details. I will go through each layer of the project

- **Client App (FE)**

```yml
client-service:
    build:
      context: ./client-app
      dockerfile: Dockerfile
    container_name: client-app-container
    ports:
      - "80:80"
    restart: always # Auto restart when it fail
```
- **Nginx**

```yml
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
```

- **Server (BE)**

```yml
 project-service:
    build:
      context: ./project-service
      dockerfile: Dockerfile
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
    build:
      context: ./user-service
      dockerfile: Dockerfile
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
    build:
      context: ./task-service
      dockerfile: Dockerfile
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
```

- **Database (MySQL)**

```yml
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
```
## Full docker-compose file

```yml
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
    build:
      context: ./project-service
      dockerfile: Dockerfile
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
    build:
      context: ./user-service
      dockerfile: Dockerfile
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
    build:
      context: ./task-service
      dockerfile: Dockerfile
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
      
  client-service: 
    build:
      context: ./client-app
      dockerfile: Dockerfile
    container_name: client-app-container
    ports:
      - "80:80"
    restart: always # Auto restart when it fail

networks:
  internal-network:
      driver: bridge
      name: internal-network

volumes:
  mysql-data: 
```
## How to run
After finished docker-compose, open terminal and run

```bash
docker compose up -d
```
It will build, create, and start the services defined in a docker-compose.yml file for you.

## Conclusion
In this post, I focused on the overall structure and main services. In an upcoming topic, I’ll dive deeper into how healthchecks, volumes, and networks work to further improve reliability and maintainability in your deployments.
Happy Coding !!!

