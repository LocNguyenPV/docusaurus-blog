# Install ArgoCD On-Premise & GCP

ArgoCD watches your Git repo and automatically syncs it to your Kubernetes cluster. Push code → ArgoCD deploys. No manual `kubectl apply` spam ever again.

---

## Prerequisites (Quick Check)

Before we start, make sure you have:

- **A running Kubernetes cluster**  
  On-premise (kubeadm, k3s, RKE2) or GKE—doesn't matter.

- **kubectl installed and connected**  
  Test with:

  ```bash
  kubectl get nodes
  ```

  You should see your nodes in `Ready` status.

- **Cluster admin access**  
  ArgoCD needs to create namespaces, CRDs, and RBAC roles.

---

## On-premis Installation

### Step 1: Install ArgoCD (One Command)

Create the ArgoCD namespace and install everything in one go:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This installs all ArgoCD components: API server, repo server, application controller, Redis, and all the required configs.

**Wait for all pods to start:**

```bash
kubectl get pods -n argocd -w
```

After 1–2 minutes, you should see all pods in `Running` status:

```
NAME                                  READY   STATUS    RESTARTS   AGE
argocd-application-controller-0       1/1     Running   0          1m
argocd-dex-server-xxxx                1/1     Running   0          1m
argocd-redis-xxxx                     1/1     Running   0          1m
argocd-repo-server-xxxx               1/1     Running   0          1m
argocd-server-xxxx                    1/1     Running   0          1m
```

Press `Ctrl+C` to stop watching once all pods are running.

---

### Step 2: Access the ArgoCD UI

By default, ArgoCD runs as a `ClusterIP` service—only accessible inside the cluster. The easiest way to access it is port-forwarding.

**Run this command:**

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Keep this terminal running. Now open your browser and go to:

```
https://localhost:8080
```

You'll see a certificate warning (ArgoCD uses a self-signed cert). Click **Advanced → Proceed** to continue.

---

### Step 3: Get the Admin Password

ArgoCD generates a random admin password on first install and stores it in a Kubernetes secret.

**Get the password:**

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Copy the output (e.g., `RGfpwaAvhnh14Krl`).

**Login credentials:**

- **Username:** `admin`
- **Password:** (the password you just copied)

Enter these in the ArgoCD login page.

---

## Installing on GKE (Google Cloud)

The install process is identical—just create a GKE cluster first, then run the same commands.

**Create a GKE cluster:**

```bash
gcloud container clusters create argocd-demo \
  --zone us-central1-a \
  --num-nodes 3
```

**Connect kubectl:**

```bash
gcloud container clusters get-credentials argocd-demo --zone us-central1-a
```

**Install ArgoCD (same command):**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**Access UI:**

If you want a public IP instead of port-forward on GKE:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Wait 2 minutes, then get the external IP:

```bash
kubectl get svc argocd-server -n argocd
```

Open `https://<EXTERNAL-IP>` in your browser.

---

## Deploy Your First App

Let's deploy a sample app from ArgoCD's official examples repo.

**Via CLI (fastest):**

```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

This creates an app but doesn't deploy it yet.

**Sync (deploy) the app:**

```bash
argocd app sync guestbook
```

ArgoCD will apply all Kubernetes manifests from the Git repo to your cluster.

**Check status:**

```bash
argocd app get guestbook
```

You should see `Health Status: Healthy` and `Sync Status: Synced`.

**Verify in Kubernetes:**

```bash
kubectl get all -n default
```

You'll see the guestbook deployment, service, and pods running.

---

### Via UI (If You Prefer Clicking)

1. In the ArgoCD UI, click **+ NEW APP**.
2. Fill in:
   - **Application Name:** `guestbook`
   - **Project:** `default`
   - **Sync Policy:** `Manual` (or `Automatic` if you want auto-sync on every Git commit)
   - **Repository URL:** `https://github.com/argoproj/argocd-example-apps.git`
   - **Revision:** `HEAD`
   - **Path:** `guestbook`
   - **Cluster URL:** `https://kubernetes.default.svc`
   - **Namespace:** `default`
3. Click **CREATE**.
4. The app appears as `OutOfSync`. Click **SYNC → SYNCHRONIZE**.
5. After a few seconds, the app turns **Healthy** and **Synced**.

---

## Common Issues (Quick Fixes)

**Pods stuck in `Pending`:**

```bash
kubectl describe pod -n argocd <pod-name>
```

Usually means not enough resources (CPU/RAM). Scale down or add nodes.

**Can't access `localhost:8080`:**

Make sure port-forward is still running in another terminal. If it stopped, run it again:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

**App stays `OutOfSync`:**

Click **SYNC** in the UI or run:

```bash
argocd app sync <app-name>
```

**Forgot admin password:**

Get it again:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

---

## That's It

You now have ArgoCD running and a sample app deployed via GitOps. From here:

- **Push changes to your Git repo** → ArgoCD auto-syncs (if you enabled automatic sync).
- **Add your own apps** by pointing ArgoCD to your repos.
- **Deploy to multiple clusters** by adding external clusters with `argocd cluster add`.
