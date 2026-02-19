# pipecd-fasttrack 

learning gitops usually costs money (cloud credits, gcp/aws). i wanted to change that.

this is a simple, zero-cost guide to getting your first project deployed on PipeCD using nothing but GitHub Codespaces and Kind. no cloud bill, no local setup drama.

### why did i build this?
when i first tried pipecd, i got stuck on some tricky connection and port-mapping issues that almost made me quit. i figured if i was struggling, other students were too. 

this repo is my attempt to lower the barrier for the next wave of pipecd contributors. i've documented the fixes for the common "connection traps" so you can get straight to the fun part: shipping code.

## 1. Cluster Setup

Start with a clean Kind cluster.

```bash
kind create cluster --name pipecd-lab

```

## 2. Install Control Plane

This puts the server and UI into the cluster.

```bash
curl -sL https://pipecd.dev/install.sh | sh

```

Check that pods are running: `kubectl get pods -n pipecd`.

## 3. Configure the Piped Agent

The agent is the part that watches your Git repo.

### Get your ID and Key

1. Open the PipeCD UI (usually on port 8080).
2. Go to **Settings** -> **Piped** tab.
3. Click **+ADD**. Give it a name like `codespace-agent`.
4. **Copy the ID and the Secret Key immediately.** You won't see the key again.

### Prep the Secret

Base64 the key (use -n to avoid newline errors):

```bash
echo -n "PASTE_YOUR_SECRET_KEY_HERE" | base64

```

### Deploy the Agent

Run this command. Replace `YOUR_ID` and `YOUR_BASE64_KEY` with the ones you just got. I changed the port to 9080 because 8080 throws 404 errors.

```bash
curl -s https://raw.githubusercontent.com/pipe-cd/pipecd/master/quickstart/manifests/piped.yaml | \
sed "s/<YOUR_PIPED_ID>/YOUR_ID/g; \
s/<YOUR_PIPED_KEY_DATA>/YOUR_BASE64_KEY/g; \
s/apiAddress: pipecd:8080/apiAddress: pipecd-server:9080/g" | \
kubectl apply -n pipecd -f -

```

## 4. App Manifests

Create these in `apps/web/` in your repo and **push them to GitHub**.

**deploy.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80

```

**.pipe.yaml**
(The dot in the filename is mandatory)

```yaml
apiVersion: pipecd.dev/v1beta1
kind: KubernetesApp
spec:
  pipeline:
    stages:
      - name: K8S_CANARY_ROLLOUT
        with:
          replicas: 1
      - name: WAIT_APPROVAL
      - name: K8S_PRIMARY_ROLLOUT
      - name: K8S_CANARY_CLEAN

```

## 5. Connecting your Repo

1. In the UI, go to **Settings** -> **Repositories** tab.
2. Click **+ADD**.
3. **Repository ID:** `lab-repo`
4. **Remote URL:** Your GitHub repo URL.
5. **Branch:** `main`
6. Click **Save**.

## 6. Deployment & Sync

1. Go to **Applications** in the sidebar. Click **+ADD**.
2. **Name:** `my-app`
3. **Piped:** Select the agent you made in Step 3.
4. **Repository:** Select `lab-repo`.
5. **Path:** `apps/web`
6. **Config Filename:** `.pipe.yaml`
7. Click **Save**, then click the **SYNC** button on the app card.

## How to know you are done

1. **The Green Check:** The app status in the UI changes to **SYNCED**.
2. **The Pipeline:** Click the app name. You should see a pipeline waiting at `WAIT_APPROVAL`. Click **Approve**.
3. **The Cluster:** Run `kubectl get pods`. You should see `web-app` pods running.

## Troubleshooting

* **CrashLoopBackOff:** Check logs (`kubectl logs -n pipecd deployment/piped`). If it says "illegal base64", re-do the secret encoding with `echo -n`.
* **404 Errors:** Means the agent is hitting the gateway instead of the server. Use `pipecd-server:9080`.
* **Config Not Found:** Make sure you pushed `.pipe.yaml` to GitHub. If it's only on your computer, PipeCD can't see it.

  I'll drop a vid and a blog on this soon!
  

