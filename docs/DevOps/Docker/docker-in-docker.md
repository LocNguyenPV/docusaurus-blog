---
sidebar_position: 5
---

# Docker-in-Docker (DinD)
## What is DinD?
For a simple terms, I will use docker to run another docker in docker 🤪 

![DinD](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bdvgrut48ktkmfsf3anr.png)
<figCaption>DinD Methodology</figCaption>

## Why is it?
It can be used in many cases, but I use it to simulate a multi-host environment on a single physical machine.

## How to do it?
Open your command line and run it
```bash
docker run --privileged -d --name <your-docker-name> docker:dind 
```
That's it, now I have a "virtual" of virtual machine to deploy and run the image

**Warning ❗❗❗:** Because it run with `--privileged`, so it's only use for dev/test environment => **Never use in production**

