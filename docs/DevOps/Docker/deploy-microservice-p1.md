---
sidebar_position: 3
---

# Deploy microservice with Docker from scratch - P1
In this series, I will build and deploy a simple microservice app with docker. Everything begin from scratch 😎

## Prerequisite
- Installed Docker (If you don't have it, follow the instructions for [Windows](https://dev.to/locnguyenpv/install-docker-on-window-5cg0) or [Linux](https://dev.to/locnguyenpv/install-docker-on-linux-2k1k))

## Tech stack
Right here, I will use:
- **FE**: Angular
- **BE**: Java
- **DB**: MySQL

**P/s:** You can use your preferred stack, but scripting a `docker-image` build may differ slightly.


## Architecture

![Architecture](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3832jj6nut8b7q167le1.gif)
<figCaption>Architecture</figCaption>

The system will have 6 components:
1) **Client app:** Interact BE via API 
2) **Nginx (gateway):** Reverse proxy and load balancer
3) **Project Service:** Handle request relate to project
4) **Task Service:** Handle request relate to task
5) **User Service:** Handle request relate to user
6) **Database Layer (MySQL):** Data persistence

## Project Structure
```plaintext
/
├── client-app/ 
│   ├── src/
│   └── Dockerfile 
├── project-service/
│   ├── src/ 
│   └── Dockerfile 
├── task-service/ 
│   ├── src/
│   └── Dockerfile
├── user-service/ 
│   ├── src/
│   └── Dockerfile
├── docker-compose
└── nginx.conf
```
**❗Important thing:** Did you notice every service have each own <u>Dockerfile</u> 🤔 

## Docker file
- FE (AngularJS)
```yml
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
COPY angular*.json ./
COPY tsconfig*.json ./
RUN npm install --legacy-peer-deps
COPY . .
RUN npm run build -- --configuration production

FROM nginx:alpine AS production
COPY --from=builder /app/dist/client-app/browser/ /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

- BE (Java)

```yml
FROM maven:latest AS build
WORKDIR /projectservice
COPY . .
RUN mvn clean package -DskipTests

FROM eclipse-temurin:21-jdk-alpine
WORKDIR /projectservice
COPY --from=build /projectservice/target/*.jar app.jar
EXPOSE 8081
ENTRYPOINT ["java", "-jar", "/projectservice/app.jar"]
```
As mentioned above, each service will have its own Dockerfile so remember <u>clone and update</u> it for each service.

## Conclusion 
Okay, I have an overview of the project and its functionality. In the [next post](https://dev.to/locnguyenpv/deploy-microservice-with-docker-from-scratch-p2-4hal), I'll delve into implementation details. 

Happy Coding!!!

