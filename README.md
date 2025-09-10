Deploying a Django App to Kubernetes (kind) — End-to-End
1) Prerequisites & What We’re Building
Goal: containerize a Django application, push the image to Docker Hub, and run it on a local Kubernetes cluster created with kind (Kubernetes-in-Docker).

We’ll create a dedicated namespace, a Deployment for the Django app, and a Service to expose it inside the cluster.

For local access, we’ll use kubectl port-forward to forward traffic from our machine to the Service.

This approach is ideal for learning Kubernetes primitives end-to-end without any cloud costs.

You can later swap kind for a managed cluster (EKS/GKE/AKS) with minimal changes.


2) Install Docker
Docker is required to build and run container images. Make sure Docker Desktop or Docker Engine is installed and running.

Verify installation with docker version and docker info — both should return successfully.

Enable Linux containers (default on Linux). On Windows/Mac, Docker Desktop handles VM setup automatically.

Confirm you can pull images: docker pull hello-world and then docker run hello-world.

(Optional) Log in to Docker Hub now to avoid prompts later: docker login.


3) Install kubectl
kubectl is the Kubernetes CLI; we’ll use it to create namespaces, apply manifests, and inspect resources.

Install from your OS package manager or the official docs; then verify with kubectl version --client.

Set a default output preference: kubectl get pods -A should work (even if no clusters are configured yet).

kubectl talks to your current cluster via a kubeconfig file (we’ll generate this when we create a kind cluster).

Keep kubectl updated — client/server minor versions should be reasonably close for best compatibility.


4) Install kind & Create a Local Kubernetes Cluster
kind runs Kubernetes nodes as Docker containers — perfect for local development and demos.

Install kind (binary or via go install) and verify with kind version.

Create a cluster:

 kind create cluster --name=tws-cluster

kind will download a node image, start a control-plane, and set your kubeconfig context automatically.

Confirm the cluster is up: kubectl cluster-info and kubectl get nodes should show a control-plane (and workers if you used a multi-node config).


5) Clone the Django Application from GitHub
Pick your Django repository (public or private); for example:

 git clone https://github.com/<your-user>/<your-django-repo>.git
cd <your-django-repo>

Ensure the project has a manage.py, requirements.txt, and a Django settings module.

(Optional) Create a Python virtualenv for local testing if you want to run it outside Docker.

Verify dependencies in requirements.txt include Django and any DB drivers (e.g., mysqlclient/psycopg2).

Make sure the app runs locally first if you’re unsure: python manage.py runserver.


6) Write a Production-Aware Dockerfile for Django
Here’s a simple, solid Dockerfile that runs the dev server and binds to 0.0.0.0:8000 (critical for Kubernetes):

 FROM python:3.9

WORKDIR /app/backend

# Install system deps (only if you need MySQL; remove if not)
RUN apt-get update \
    && apt-get install -y gcc default-libmysqlclient-dev pkg-config \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt /app/backend/
RUN pip install --no-cache-dir -r requirements.txt

COPY . /app/backend

EXPOSE 8000
# Simple entrypoint (for demo). For prod, prefer Gunicorn + migrations.
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]

Key detail: runserver 0.0.0.0:8000 so the process listens on all interfaces inside the container.

If you use a DB (e.g., MySQL), inject env vars at runtime (not hard-coded here).

For production, switch to Gunicorn and run migrate/collectstatic via a startup script or init containers.


7) Build the Image & Test Locally (Optional)
Build the image and tag it with your Docker Hub name:

 docker build -t mdekramuddin/django-app-image:latest .

Confirm the image exists: docker images | grep django-app-image.

(Optional) Run locally to sanity-check:

 docker run -p 8000:8000 mdekramuddin/django-app-image:latest

If the app serves on http://localhost:8000, your container is healthy.

Stop the container when done (Ctrl+C or docker stop <id>).


8) Push the Image to Docker Hub
Log in if you haven’t: docker login.

Push your image:

 docker push mdekramuddin/django-app-image:latest

Using explicit version tags (e.g., :v1, :v2) makes rollbacks and upgrades clearer.

You can keep :latest for “tip of main” and versioned tags for releases.

Verify the image is visible on your Docker Hub repository page.


9) Create a Dedicated Kubernetes Namespace
Namespaces isolate resources and keep your cluster tidy.

 kubectl create namespace notes-app

Confirm it exists: kubectl get ns.

You can set your default namespace for all commands via kubeconfig or context if you prefer.

Keeping app resources in one namespace also makes cleanup simple.

I’ll use -n notes-app in the commands below for clarity.


10) Create the Django Deployment (Pods Managed by a ReplicaSet)
A Deployment ensures the desired number of replicas (pods) are running and handles rolling updates.

Save this as deployment.yml:

 apiVersion: apps/v1
kind: Deployment
metadata:
  name: notes-app-deployment
  namespace: notes-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: notes-app
  template:
    metadata:
      labels:
        app: notes-app
    spec:
      containers:
        - name: notes-app
          image: mdekramuddin/django-app-image:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8000
          # Uncomment if your app uses a DB and expects env vars:
          # env:
          #   - name: DB_NAME
          #     value: notesdb
          #   - name: DB_USER
          #     value: root
          #   - name: DB_PASSWORD
          #     value: mypassword
          #   - name: DB_HOST
          #     value: mysql-service
          #   - name: DB_PORT
          #     value: "3306"

Apply it: kubectl apply -f deployment.yml.

Watch the rollout: kubectl rollout status deployment/notes-app-deployment -n notes-app.

Verify pod creation: kubectl get pods -n notes-app.

If there’s any crash, kubectl logs <pod> -n notes-app will show why.


11) Create a ClusterIP Service for the App
A Service gives your pods a stable, discoverable virtual IP and DNS name.

We’ll use a ClusterIP (default) to expose the app inside the cluster; we’ll port-forward from our machine to this Service.

Save as service.yml:

 apiVersion: v1
kind: Service
metadata:
  name: notes-app-service
  namespace: notes-app
spec:
  selector:
    app: notes-app
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 8000
  type: ClusterIP

Apply with kubectl apply -f service.yml.

Check it exists: kubectl get svc -n notes-app.

You should see notes-app-service with PORT(S) 8000/TCP.


12) (Optional) Add a MySQL Database In-Cluster
If your Django settings use MySQL, deploy a DB inside the same namespace so the app can connect via service DNS.

Minimal manifest mysql.yml (for learning/demo — add PVC/secrets for real use):

 apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: notes-app
spec:
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: notes-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: mypassword
            - name: MYSQL_DATABASE
              value: notesdb
          ports:
            - containerPort: 3306

Apply it: kubectl apply -f mysql.yml.

Confirm: kubectl get pods,svc -n notes-app | grep mysql.

Update your Django Deployment env to point at mysql-service:3306 (see commented env block in step 10).

(Tip) For reliability, add an initContainer to wait until MySQL is ready:

 initContainers:
  - name: wait-for-mysql
    image: busybox:1.36
    command: ['sh','-c','until nc -z mysql-service 3306; do echo waiting for mysql; sleep 2; done']


13) Apply Manifests & Verify Everything
Apply all manifests (order doesn’t strictly matter, but I do Deployment then Service):

 kubectl apply -f deployment.yml
kubectl apply -f service.yml
# if using DB:
# kubectl apply -f mysql.yml

Check namespace resources: kubectl get all -n notes-app.

Describe your pod if anything looks off: kubectl describe pod <pod> -n notes-app.

Logs are your friend: kubectl logs -f <pod> -n notes-app (look for “Starting development server at http://0.0.0.0:8000/”).

If the pod shows CrashLoopBackOff, read logs; fix app config (DB env, bindings, migrations) and re-apply.


14) Port-Forward the Service & Access in Browser
With a ClusterIP Service, use port-forward to reach it from your machine:

 kubectl port-forward service/notes-app-service -n notes-app 8000:8000 --address=0.0.0.0

Leave this terminal running; it forwards local port 8000 to the Service’s 8000 in the cluster.

Now open http://localhost:8000 in your browser — you should see your Django app.

If you get “connection refused,” the container likely isn’t listening (ensure runserver 0.0.0.0:8000).

If you use a DB, ensure it’s reachable; add the initContainer wait if startup order causes issues.


15) Rolling Updates (Rebuild → Push → Roll Out)
After code changes, rebuild and push a new tag:

 docker build -t mdekramuddin/django-app-image:v2 .
docker push mdekramuddin/django-app-image:v2

Update the Deployment’s image:

 kubectl set image deployment/notes-app-deployment notes-app=mdekramuddin/django-app-image:v2 -n notes-app

Watch rollout: kubectl rollout status deployment/notes-app-deployment -n notes-app.

If something breaks, you can undo: kubectl rollout undo deployment/notes-app-deployment -n notes-app.

Prefer semantic tags (v1, v2) over latest for traceability.


16) Troubleshooting Cheatsheet
Port-forward says “connection refused” → container isn’t listening on 0.0.0.0:8000 or app crashed before binding. Check kubectl logs.

CrashLoopBackOff with DB errors → inject correct DB env vars; ensure DB Service exists; add initContainer to wait for DB.

Image changes not taking effect → push a new tag and update Deployment; or set imagePullPolicy: Always.

Pod Pending → check PVC/PV binding or node resources: kubectl describe pod <pod>.

Service not selecting pods → labels must match between Service spec.selector and Pod metadata.labels.


17) Cleanup (Optional)
Delete app resources:

 kubectl delete -f service.yml
kubectl delete -f deployment.yml
# if used:
# kubectl delete -f mysql.yml

Delete the namespace: kubectl delete ns notes-app.

Delete the kind cluster when done: kind delete cluster --name=tws-cluster.

Remove local images/containers if you’re freeing disk space.

Keep your manifests under version control for easy re-deploys.


18) Results & What I Learned (great for LinkedIn)
✅ Containerized a Django app with a clear Dockerfile and best-practice binding to 0.0.0.0:8000.

✅ Stood up a local Kubernetes cluster using kind, keeping everything reproducible and free.

✅ Deployed with a Deployment + Service and accessed the app via kubectl port-forward.

✅ (Optional) Added a MySQL Deployment/Service and used env vars + an initContainer to handle startup order.

✅ Practiced rolling updates with versioned Docker tags and kubectl rollout, plus learned key debugging commands.
