---
sidebar_position: 2
---

# YAML - Kubernetes 'Favorite' Configuration Language

When work with <u>Kubernetes</u>, we can interact with it via UI (I'll write about it in future) or configuration languages (YAML, JSON or HCL). This post will focus on **YAML - Yet Another Markup Language**.

![YAML meme](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2n5iykjkwqg0b00pzkbi.png)



## Structure 

When we import a YAML file in Kubernetes, we create a Kubernetes object which is a persistent entity in the cluster. Examples:

- Pod - the place where you run your application
- Service - a way to expose your app
- Deployment - manage replicas and update
- ConfigMap, Secret, Ingress, etc. 

We just need focus to **four primary top-level fields** in YAML

- **apiVersion:** Specifies the version of the Kubernetes API that the object belongs to. For example:
       - **apps/v1** for <u>Deployments </u>
       - **v1** for <u>Pods </u> and <u>Services</u>.

- **kind:** Defines the type of Kubernetes object being created. Common examples include `Pod`, `Deployment`, `Service`, `ReplicaSet`, `Ingress`, etc.

- **metadata:** Contains data that identifies the object within the cluster and provides descriptive information. This typically include:
       - **name:** unique name for object (require)
       - **namespace:** namespace the object belong to, default namespace is `default` (optional)
       - **labels:** Attach label for object, it help another object can select base on this
       - **annotation:** Often used for tools or specific configurations.

- **spec:** Object configuration. The content of this section depending on the kind of object. For example:
       - For a **Deployment** kind, spec defines <u>the number of replicas</u>, <u>the pod template</u> (including container images, ports, and resource limits), <u>update strategy</u>, and <u>selectors</u>.
       - For a **Service** kind, spec defines the <u>port mappings</u>, <u>selector </u> to find target pods, and <u>type of service</u> (e.g., ClusterIP, NodePort, LoadBalancer).
       - For a **Pod** kind, spec defines the containers within the pod, volumes, restart policy, etc.

Example of YAML in deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  generation: 2
  labels:
    app: wordpress #EX: app:front-end
  name: wordpress
  namespace: wp-ecommerce
spec:
  progressDeadlineSeconds: 600
  replicas: 2
  revisionHistoryLimit: 11
  selector:
    matchLabels:
        app: wordpress #EX: app:front-end
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: wordpress  # ✅ This matches the selector
    spec:
      containers:
        - image: wordpress:latest
          imagePullPolicy: Always
          name: wordpress
          ports:
            - containerPort: 80
              name: tcp
              protocol: TCP
      restartPolicy: Always
```
If you want run it in Kubernetes, just run in your **master node**
```bash
kubectl apply -f your-yaml-file.yaml
```

## Conclusion
In reality, others people can use JSON / HCL for Kubernetes too. But for some reason, maybe it look cooler 👀 (J4F), they usually use YAML for Kubernetes. Each format has its pros and cons, use what you're comfortable with.

![yaml meme 2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0zp4jmnubbb24ij9yg2r.png)

Happy coding !
