---
sidebar_position: 1
---

# Install Docker on Window

## 1) Install WSL 2 (if not install yet)

- **Window 11:** already have `WSL 2`

- **Window 10 and below:** Open <u>PowerShell</u> with Administrator role and run

```bash
wsl --install
```

## 2) Install Docker Desktop

- Access this [URL](https://www.docker.com/products/docker-desktop/)
- Choose “Download for Windows (WSL 2)”
- Download and install everything you need
- After installed, reboot your machine to get it done

## 3) Check docker version

Open Terminal and run

```bash
docker --version
```

It will show something like `Docker version x.x.x, build ....`

## 4) Run and test docker

Open Terminal and run

```bash
docker run hello-world
```

If everything work fine, you'll see

> Hello from Docker!

This message shows that your installation appears to be working correctly

Now you can start your journey with Docker!

Happy Coding.
