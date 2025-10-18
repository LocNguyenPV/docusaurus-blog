---
sidebar_position: 3
---

# Pod - The smallest unit in Kubernetes
I have gone through Kubernetes architecture in the past, you can go over [here](https://dev.to/locnguyenpv/kubernetes-architecture-overview-mll). In that article, we know Kubernetes run application in the place call **Pod**


![meme](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/at9n7dmbquwdp350xnqw.png)

## What is Pod?
Pod is the smallest unit in Kubernetes for deploy and run an application. Pod can contain one or many containers and they sharing the pod's resource for each others. 

![pod_1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/lerx3h8r3lwlairkmoul.png)
<figCaption>Pod example</figCaption>


## Deploy the first app 
I would use simple image `okteto/hello-world:node` for test. Or you can deploy custom image to docker hub by yourself, I have written in [this article](https://dev.to/locnguyenpv/multi-host-deployment-and-management-using-portainer-110l)

```yaml
apiVersion: v1 
kind: Pod # Declare Pod resource
metadata:
  name: hello-pod # The name of the pod
spec:
  containers:
    - image: okteto/hello-world:node # Image to create the container
      name: hello-pod # The name of the container
      ports:
        - containerPort: 3000 # The port the app is listening on 
          protocol: TCP
```

Access to your master node and create the file `test.yaml` with content above. After that, run 

```bash
kubectl apply -f test.yaml
```
Check it deploy success or not by the command
```bash
kubectl get pod
```
It must show the pod with name `hello-pod` and `status` is running 

![first app](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5du3lk8kr0soyewl6hov.png)

If `status` is `ContainerCreating`, wait for a few minutes and try again because the container is being create. To ensure the image runs correctly, I will use a network proxy for expose the port. Currently, here is my app

![pod_status](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/dgh5toplw7cukbjjebl6.png)

Run the command 
```bash
kubectl port-forward pod/hello-pod 3000:3000 
```
It will expose port `3000`, listening and coordinating access to the container app on the same port. 

![port expose](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fhxdx5rmibxuutnulxt6.png)

Test it with `curl localhost:3000`, if you receive <u>hello message</u>...You have your first app with Kubernetes 🎉


![award](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/cxztn5jdvfaqpigh06oo.png)

## Conclusion
We have deploy and run simple app with pod in Kubernetes. In reality, no one deploy and run app like this 🤣, they usually use <u>Deployment</u>, <u>StatefulSet</u>, <u>DaemonSet</u>. But that is another article, the goal of this post is to show you what a Pod is and its intend. 

In the next article, I will write how to group and manage pod by **label** and **namespace**

Happy Coding!

