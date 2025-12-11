# Run first job: Hello CI

## Option 1: Freestyle job

- Choose **New Item**
  ![alt text](./img/first-pipline/image.png)
- Choose **Freestyle project**.
  ![alt text](./img/first-pipline/image-1.png)
- Add build step with “Execute shell”:
  ![alt text](./img/first-pipline/image-2.png)
- Fill out with following script
  ```bash
  echo "Hello Jenkins from Docker"
  ```
- Click save and choose **Build Now**. If console log show like image below, it's mean Jenkins work fine.
  ![alt text](./img/first-pipline/image-3.png)

### Option 2: Pipeline job

- Choose **New Item**
  ![alt text](./img/first-pipline/image.png)

- Choose **pipeline**
  ![alt text](./img/first-pipline/image-4.png)

- Config git/gitlab to build
  ![alt text](./img/first-pipline/image-5.png)

- Click save and choose **Build Now**
