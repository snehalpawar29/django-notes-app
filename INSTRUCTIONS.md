# 📘 Django Notes App — Deployment & Operations Instructions

This document contains the detailed operational guide for the project. The main `README.md` intentionally stays short and points here for step-by-step instructions.

---

# 1. 🧱 Architecture Overview

The application runs with this structure:

```text
Internet
   │
   ▼
AWS EC2 Public IP :30080
   │
   ▼
Docker Engine
   │
   ▼
Kind Kubernetes Cluster
   │
   ├── Django Deployment
   │      └── Django Pod :8000
   │
   └── MySQL Deployment
          └── MySQL Pod :3306
                 │
                 ▼
            PVC → PV
                 │
                 ▼
          /mnt/data/mysql
```

Django reaches MySQL through the Kubernetes service name:

```text
mysql-service
```

Inside the namespace, Kubernetes DNS resolves it as:

```text
mysql-service.django-ns.svc.cluster.local
```

---

# 2. ✅ Prerequisites

On the AWS EC2 Ubuntu machine, install/check:

```bash
docker --version
kubectl version --client
kind version
git --version
```

The EC2 instance must have enough CPU/RAM for Docker and the Kind cluster.

---

# 3. 📦 Project Structure

```text
django-notes-app/
│
├── k8s/
│   ├── cluster.yml
│   ├── namespace.yml
│   ├── deployment.yml
│   ├── service.yml
│   ├── db_deployment.yml
│   ├── db-svc.yml
│   ├── db-pv.yml
│   └── db-pvc.yml
│
├── screenshots/
│   └── architecture.png
│
├── README.md
└── INSTRUCTIONS.md
```

---

# 4. 🐳 Docker Image

The Django application image used by the Kubernetes deployment is:

```text
snehalpawar2945/notes-app
```

Check the image:

```bash
docker images
```

Pull it manually if required:

```bash
docker pull snehalpawar2945/notes-app
```

---

# 5. ☸️ Create the Kind Cluster

The project uses a Kind cluster named:

```text
django-cluster
```

The important port mapping is:

```text
EC2 host :30080
        ↓
Kind control-plane container :8000
```

Create the cluster:

```bash
cd ~/django-notes-app/k8s

kind create cluster --config cluster.yml
```

Verify:

```bash
kind get clusters
kubectl get nodes -o wide
docker ps
```

Expected cluster nodes include:

```text
django-cluster-control-plane
django-cluster-worker
```

---

# 6. 🏷️ Create Namespace

```bash
kubectl apply -f namespace.yml
```

Verify:

```bash
kubectl get namespace
```

The application namespace is:

```text
django-ns
```

---

# 7. 💾 Configure MySQL Persistent Storage

The project uses:

```text
PersistentVolume
       ↓
PersistentVolumeClaim
       ↓
MySQL /var/lib/mysql
       ↓
EC2 hostPath /mnt/data/mysql
```

Create the host directory on the EC2 machine if necessary:

```bash
sudo mkdir -p /mnt/data/mysql
sudo chmod 777 /mnt/data/mysql
```

Apply the PV and PVC:

```bash
kubectl apply -f db-pv.yml
kubectl apply -f db-pvc.yml
```

Verify:

```bash
kubectl get pv
kubectl get pvc -n django-ns
```

Expected relationship:

```text
mysql-pvc  →  mysql-pv
```

The PV uses:

```text
Storage: 1Gi
Access mode: ReadWriteOnce
Storage class: manual
Reclaim policy: Retain
Host path: /mnt/data/mysql
```

> ⚠️ Important for Kind: `hostPath` refers to a path on the node where the volume is mounted. With multi-node Kind, node placement matters. For a production deployment, prefer a cloud-native storage class instead of a local `hostPath`.

---

# 8. 🗄️ Deploy MySQL

Apply:

```bash
kubectl apply -f db_deployment.yml
```

Create/check the MySQL service:

```bash
kubectl apply -f db-svc.yml
```

Check:

```bash
kubectl get pods -n django-ns
kubectl get svc -n django-ns
```

Wait until MySQL is ready:

```bash
kubectl get pods -n django-ns -w
```

Check MySQL logs:

```bash
kubectl logs deployment/db-deployment -n django-ns
```

A healthy MySQL instance should eventually report that it is ready for connections on port `3306`.

---

# 9. 🛢️ Create the Django Database

The Django application is configured to use:

```text
Database: test_db
User: root
Password: root
Host: mysql-service
Port: 3306
```

Create the database:

```bash
kubectl exec -it deployment/db-deployment -n django-ns -- \
mysql -uroot -proot -e "CREATE DATABASE test_db;"
```

If it already exists, MySQL will report an error. That is not a problem.

Safer version:

```bash
kubectl exec -it deployment/db-deployment -n django-ns -- \
mysql -uroot -proot -e "CREATE DATABASE IF NOT EXISTS test_db;"
```

Verify:

```bash
kubectl exec -it deployment/db-deployment -n django-ns -- \
mysql -uroot -proot -e "SHOW DATABASES;"
```

---

# 10. 🚀 Deploy Django

Apply the application deployment:

```bash
kubectl apply -f deployment.yml
```

Check:

```bash
kubectl get pods -n django-ns -o wide
```

The Django pod should become:

```text
1/1 Running
```

Check logs:

```bash
kubectl logs deployment/django-deployment -n django-ns
```

---

# 11. 🔄 Run Django Migrations

After the MySQL database exists and the Django pod is running:

```bash
kubectl exec -it deployment/django-deployment -n django-ns -- \
python manage.py migrate
```

Verify the database tables:

```bash
kubectl exec -it deployment/db-deployment -n django-ns -- \
mysql -uroot -proot test_db -e "SHOW TABLES;"
```

The project should create Django tables such as:

```text
django_migrations
auth_user
django_session
django_admin_log
```

and the Notes application's table:

```text
api_note
```

---

# 12. 🔌 Configure Django Service

The application service should select the Django pod using:

```yaml
selector:
  app: notes-app
```

The Django container listens on:

```text
8000
```

For external access through the EC2 instance, the service uses:

```text
NodePort: 30080
```

Apply it:

```bash
kubectl apply -f service.yml
```

Verify:

```bash
kubectl get svc backend-service -n django-ns
```

For the NodePort setup, the expected output contains:

```text
8000:30080/TCP
```

Check endpoints:

```bash
kubectl get endpoints backend-service -n django-ns
```

You should see a Django pod endpoint similar to:

```text
10.244.x.x:8000
```

---

# 13. 🌐 EC2 Public Access

The intended URL is:

```text
http://EC2_PUBLIC_IP:30080
```

For example:

```text
http://YOUR_EC2_PUBLIC_IP:30080
```

Do not use the Kubernetes ClusterIP from outside the cluster.

The traffic path is:

```text
EC2 :30080
   ↓
Kind port mapping
   ↓
NodePort :30080
   ↓
Django Service
   ↓
Django Pod :8000
```

---

# 14. 🔐 AWS Security Group

If the website cannot be reached from your browser, check the EC2 Security Group.

Add an inbound rule for:

```text
Type: Custom TCP
Port: 30080
Source: your IP address / required CIDR
```

For temporary testing, a broader source may be used, but restricting the source to your own IP is safer.

Also verify Ubuntu's firewall if enabled:

```bash
sudo ufw status
```

---

# 15. 🩺 Troubleshooting

## Django pod is running but website is unavailable

Check:

```bash
kubectl get svc backend-service -n django-ns
kubectl describe svc backend-service -n django-ns
kubectl get endpoints backend-service -n django-ns
```

The service must have an endpoint.

If there is no endpoint, check the labels:

```bash
kubectl get pods -n django-ns --show-labels
```

The Django pod must contain:

```text
app=notes-app
```

---

## Port-forward gives `connection refused`

If you run:

```bash
kubectl port-forward deployment/django-deployment 8000:8000 -n django-ns
```

and get:

```text
connect: connection refused
```

check whether Django is actually listening on `0.0.0.0:8000`:

```bash
kubectl exec -it deployment/django-deployment -n django-ns -- \
ps aux
```

The process should contain:

```text
runserver 0.0.0.0:8000
```

For the EC2 + Kind NodePort architecture, use the NodePort path instead of relying on `kubectl port-forward`.

---

## Django says `Unknown server host 'mysql-service'`

First check DNS:

```bash
kubectl exec -it deployment/django-deployment -n django-ns -- \
getent hosts mysql-service
```

Then check TCP connectivity:

```bash
kubectl exec -it deployment/django-deployment -n django-ns -- \
python -c "import socket; print(socket.create_connection(('mysql-service',3306),5))"
```

Check the MySQL service:

```bash
kubectl get svc mysql-service -n django-ns
```

Check its endpoints:

```bash
kubectl get endpoints mysql-service -n django-ns
```

A healthy result should contain a MySQL pod IP on port `3306`.

---

## Django connects but notes are not displayed

Check the database:

```bash
kubectl exec -it deployment/db-deployment -n django-ns -- \
mysql -uroot -proot test_db -e "SHOW TABLES;"
```

Then:

```bash
kubectl exec -it deployment/db-deployment -n django-ns -- \
mysql -uroot -proot test_db -e "SELECT * FROM api_note;"
```

If `api_note` contains records, the data has reached MySQL.

---

## MySQL pod restarts

Check:

```bash
kubectl describe pod -l app=mysql -n django-ns
kubectl logs deployment/db-deployment -n django-ns
kubectl get pvc -n django-ns
```

Also check storage:

```bash
kubectl get pv
kubectl get pvc -n django-ns
```

---

# 16. 🔎 Useful Kubernetes Commands

### All resources

```bash
kubectl get all -n django-ns
```

### Pods

```bash
kubectl get pods -n django-ns -o wide
```

### Services

```bash
kubectl get svc -n django-ns
```

### Persistent volumes

```bash
kubectl get pv
```

### Persistent volume claims

```bash
kubectl get pvc -n django-ns
```

### Deployments

```bash
kubectl get deployments -n django-ns
```

### ReplicaSets

```bash
kubectl get rs -n django-ns
```

### Detailed pod information

```bash
kubectl describe pod <POD_NAME> -n django-ns
```

### Django logs

```bash
kubectl logs deployment/django-deployment -n django-ns
```

### MySQL logs

```bash
kubectl logs deployment/db-deployment -n django-ns
```

---

# 17. 🔁 Restart Deployments

Django:

```bash
kubectl rollout restart deployment/django-deployment -n django-ns
```

MySQL:

```bash
kubectl rollout restart deployment/db-deployment -n django-ns
```

Check rollout:

```bash
kubectl rollout status deployment/django-deployment -n django-ns
kubectl rollout status deployment/db-deployment -n django-ns
```

> Kubernetes Services do not support `kubectl rollout restart`. If a Service manifest changes, use `kubectl apply -f service.yml`.

---

# 18. 🧹 Cleanup

Delete the application:

```bash
kubectl delete -f service.yml
kubectl delete -f deployment.yml
```

Delete MySQL:

```bash
kubectl delete -f db-svc.yml
kubectl delete -f db_deployment.yml
```

Delete storage resources:

```bash
kubectl delete -f db-pvc.yml
kubectl delete -f db-pv.yml
```

Delete the namespace:

```bash
kubectl delete -f namespace.yml
```

Delete the Kind cluster:

```bash
kind delete cluster --name django-cluster
```

> Because the PV uses `Retain`, deleting Kubernetes resources does not necessarily remove the data under `/mnt/data/mysql`. Handle that directory separately if you intentionally want to delete the stored database data.

---

# 19. 🧪 Final Health Check

Run these commands before considering the deployment complete:

```bash
kind get clusters

kubectl get nodes

kubectl get pods -n django-ns

kubectl get svc -n django-ns

kubectl get pv

kubectl get pvc -n django-ns

kubectl get endpoints backend-service -n django-ns

kubectl get endpoints mysql-service -n django-ns
```

Then verify Django → MySQL:

```bash
kubectl exec -it deployment/django-deployment -n django-ns -- \
python -c "import socket; print(socket.create_connection(('mysql-service',3306),5))"
```

Finally test:

```text
http://EC2_PUBLIC_IP:30080
```

---

# 20. 📌 Important Project Notes

- `backend-service` is the external-facing NodePort service.
- `mysql-service` should remain internal to the Kubernetes cluster.
- Django uses `mysql-service` as the database host.
- Django listens on container port `8000`.
- MySQL listens on container port `3306`.
- MySQL data is stored through `mysql-pvc` → `mysql-pv`.
- The Kind cluster is running inside Docker on AWS EC2.
- EC2 port `30080` is mapped into the Kind cluster.
- AWS Security Group configuration is required for external access.
- For production workloads, replace the local `hostPath` PV with durable cloud storage such as an AWS-backed storage solution.
