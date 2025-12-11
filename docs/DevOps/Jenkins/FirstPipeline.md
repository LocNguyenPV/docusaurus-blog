# Run first job: Hello CI

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
