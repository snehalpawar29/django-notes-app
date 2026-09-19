# 📝 Django Notes App — Docker + Kubernetes (Kind) on AWS EC2

[![Django](https://img.shields.io/badge/Django-4.x-092E20?logo=django&logoColor=white)](https://www.djangoproject.com/)
[![Python](https://img.shields.io/badge/Python-3.9-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1?logo=mysql&logoColor=white)](https://www.mysql.com/)
[![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-Kind-326CE5?logo=kubernetes&logoColor=white)](https://kind.sigs.k8s.io/)
[![AWS EC2](https://img.shields.io/badge/AWS-EC2-FF9900?logo=amazonaws&logoColor=white)](https://aws.amazon.com/ec2/)

> A containerized Django Notes application deployed on a **Kind Kubernetes cluster running inside Docker on an AWS EC2 Ubuntu instance**, with MySQL persistence using Kubernetes PV and PVC.

---

## 🚀 Project Overview

This project demonstrates how to containerize a Django-based application and deploy it on Kubernetes using **Kind (Kubernetes IN Docker)** running on an AWS EC2 instance.

The deployment includes:

- 🐍 Django application
- 🐳 Docker containerization
- ☸️ Kubernetes deployment using Kind
- 🗄️ MySQL 8.0 database
- 💾 Persistent storage using Kubernetes PV/PVC
- 🌐 NodePort-based application access
- ☁️ AWS EC2 Ubuntu as the deployment environment

The project focuses on practical experience with **containers, Kubernetes workloads, networking, persistent storage, and cloud-based deployment**.

---

## 🏗️ Architecture

![Django Notes App Architecture](screenshots/architecture.png)

### Request Flow

```text
User
  │
  ▼
AWS EC2 Public IP :30080
  │
  ▼
Kind / NodePort :30080
  │
  ▼
Django Pod :8000
  │
  ▼
mysql-service :3306
  │
  ▼
MySQL Pod
  │
  ▼
PV → PVC → /mnt/data/mysql
````

### Deployment Environment

```text
AWS EC2 Ubuntu
      │
      ▼
    Docker
      │
      ▼
 Kind Kubernetes Cluster
      │
 ┌────┴───────────────┐
 │                    │
Django              MySQL
Pod                  Pod
 │                    │
NodePort          PV + PVC
```

---

## 🧰 Technology Stack

| Category         | Technologies        |
| ---------------- | ------------------- |
| Application      | Django, Python      |
| Frontend         | React               |
| Database         | MySQL 8.0           |
| Containerization | Docker              |
| Orchestration    | Kubernetes, Kind    |
| Cloud            | AWS EC2             |
| Storage          | Kubernetes PV, PVC  |
| Networking       | ClusterIP, NodePort |
| OS               | Ubuntu              |

---

## 📁 Kubernetes Configuration

```text
k8s/
├── cluster.yml
├── namespace.yml
├── deployment.yml
├── service.yml
├── db_deployment.yml
├── db-svc.yml
├── db-pv.yml
└── db-pvc.yml
```

### Configuration Files

| File                | Purpose                                                  |
| ------------------- | -------------------------------------------------------- |
| `cluster.yml`       | Creates the Kind cluster and configures EC2 port mapping |
| `namespace.yml`     | Creates the `django-ns` namespace                        |
| `deployment.yml`    | Deploys the Django application                           |
| `service.yml`       | Exposes Django through NodePort `30080`                  |
| `db_deployment.yml` | Deploys MySQL 8.0                                        |
| `db-svc.yml`        | Provides internal MySQL access on port `3306`            |
| `db-pv.yml`         | Defines persistent storage for MySQL                     |
| `db-pvc.yml`        | Claims the persistent volume                             |

---

## 🚀 Deployment

### 1. Clone the Repository

```bash
git clone <YOUR_REPOSITORY_URL>
cd django-notes-app
```

### 2. Navigate to Kubernetes Configuration

```bash
cd k8s
```

### 3. Create the Kind Cluster

```bash
kind create cluster --config cluster.yml
```

### 4. Deploy Kubernetes Resources

```bash
kubectl apply -f namespace.yml

kubectl apply -f db-pv.yml
kubectl apply -f db-pvc.yml

kubectl apply -f db_deployment.yml
kubectl apply -f db-svc.yml

kubectl apply -f deployment.yml
kubectl apply -f service.yml
```

### 5. Verify the Deployment

```bash
kubectl get pods -n django-ns
kubectl get svc -n django-ns
kubectl get pv
kubectl get pvc -n django-ns
```

---

## 🌐 Application Access

The application is exposed through Kubernetes **NodePort 30080**.

```text
http://EC2_PUBLIC_IP:30080
```

The Kind configuration maps the EC2 host port `30080` to the Kind control-plane container port `8000`.

For external access, the AWS EC2 Security Group must allow inbound TCP traffic on port `30080`.

---

## 💾 Persistent MySQL Storage

MySQL data is stored using Kubernetes persistent storage.

```text
MySQL Pod
   │
   ▼
mysql-pvc
   │
   ▼
mysql-pv
   │
   ▼
EC2 hostPath
/mnt/data/mysql
```

The MySQL container uses:

```text
/var/lib/mysql
```

as its database storage location.

Using PV/PVC separates the database data from the container filesystem and provides persistent storage for the MySQL workload.

---

## 🔍 Kubernetes Verification

### Check All Resources

```bash
kubectl get all -n django-ns
```

### Check Persistent Volumes

```bash
kubectl get pv
kubectl get pvc -n django-ns
```

### Check MySQL Service

```bash
kubectl get endpoints mysql-service -n django-ns
```

### Check Django Logs

```bash
kubectl logs deployment/django-deployment -n django-ns
```

### Check MySQL Logs

```bash
kubectl logs deployment/db-deployment -n django-ns
```

---

## 🗄️ Database Verification

The MySQL database can be inspected from inside the database deployment:

```bash
kubectl exec -it deployment/db-deployment -n django-ns -- \
mysql -uroot -proot test_db -e "SHOW TABLES;"
```

This helps verify database connectivity and the presence of application tables.

---

## 📸 Screenshots

### Application

![Django Notes Application](screenshots/application.png)

### Kubernetes Resources

![Kubernetes Pods](screenshots/kubernetes.png)

### MySQL Persistent Storage

![MySQL Storage](screenshots/mysql.png)

---

## 📚 Documentation

For the complete deployment and troubleshooting process, see:

➡️ **[INSTRUCTIONS.md](INSTRUCTIONS.md)**

The documentation covers:

* Prerequisites
* Docker image workflow
* Kind cluster creation
* Kubernetes deployment
* MySQL setup
* PV/PVC configuration
* NodePort and EC2 networking
* Deployment verification
* Application and database logs
* Troubleshooting
* Cleanup

---

## 🧠 DevOps Concepts Practiced

This project provides hands-on practice with:

* 🐳 Docker containerization
* ☸️ Kubernetes deployments and services
* 🔗 Kubernetes service networking
* 🌐 NodePort exposure
* 💾 Persistent Volumes and Persistent Volume Claims
* 🗄️ MySQL container deployment
* ☁️ AWS EC2 deployment environment
* 🔍 Kubernetes troubleshooting and log inspection
* 🛠️ Application and database verification

---

## 🧹 Cleanup

To remove the Kubernetes resources:

```bash
kubectl delete -f service.yml
kubectl delete -f deployment.yml

kubectl delete -f db-svc.yml
kubectl delete -f db_deployment.yml

kubectl delete -f db-pvc.yml
kubectl delete -f db-pv.yml

kubectl delete -f namespace.yml
```

The Kind cluster can then be removed with:

```bash
kind delete cluster
```

---

## 🎯 Project Focus

The main objective of this project is to gain practical experience in deploying a containerized application using **Docker and Kubernetes**, while working with **persistent database storage and AWS EC2 networking**.

---

## 👩‍💻 Author

**Snehal Pawar**

Aspiring DevOps Engineer | AWS | Linux | Docker | Kubernetes | Terraform | Jenkins

* GitHub: [snehalpawar29](https://github.com/snehalpawar29)
* LinkedIn: [Snehal Pawar](https://www.linkedin.com/in/snehalpawar29/)
* Portfolio: [DevOps Portfolio](https://snehalpawar29.github.io/Snehal-Pawar-Devops-Portfolio/)
