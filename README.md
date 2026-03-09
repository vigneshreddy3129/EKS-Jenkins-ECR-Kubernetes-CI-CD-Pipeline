# 🚀 CI/CD Pipeline: Jenkins + EKS + ECR + Kubernetes

This project demonstrates a complete CI/CD pipeline using:

- Amazon EKS
- Jenkins
- Amazon ECR
- Docker
- Kubernetes (HPA + Ingress)

---

# 📌 Architecture Flow

Developer → GitHub → Jenkins → Docker Build → ECR → EKS → Kubernetes Pods

---
# Create Jenkins Server (EC2)
- 📍 AWS Console → EC2 → Launch Instance

```
AMI: Ubuntu 22
Type: t2.medium
Ports:
22
8080
80
443
```
- Connect to EC2

---
## Install Required Packages
-- Run inside Jenkins EC2

```bash
sudo apt update -y
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
docker --version

sudo apt install openjdk-17-jdk -y

curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
/usr/share/keyrings/jenkins-keyring.asc

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
/etc/apt/sources.list.d/jenkins.list

sudo apt update
sudo apt install jenkins -y

sudo systemctl start jenkins

sudo systemctl status jenkins

sudo usermod -aG docker jenkins


sudo cat /var/lib/jenkins/secrets/initialAdminPassword

```

## Login to Docker Hub on Jenkins Server
```
docker login
Username:
Password:
```

## Install plugins:
-- Run inside Jenkins EC2
- Docker
- Kubernetes
- Git
- Pipeline
- Amazon ECR

## Install Tools
-- Run inside Jenkins EC2

- AWS CLI
```bash
sudo apt install awscli -y
aws --version
 ```
- kubectl
```bash
curl -LO https://dl.k8s.io/release/v1.29.0/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```
- eksctl
```bash
curl --silent --location \
"https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" \
| tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin
```
- verify
```bash
eksctl version
kubectl version --client
```
## Configure AWS Credentials
- Run inside Jenkins EC2
```bash
aws configure
```
- enter
```
AWS Access Key
AWS Secret Key
Region
Output format
```

## Create EKS Cluster
- Run inside Jenkins EC2

```bash
eksctl create cluster \
--name three-tier-cluster \
--region ap-south-1 \
--nodegroup-name workers \
--node-type t3.medium \
--nodes 2 \
--nodes-min 2 \
--nodes-max 5 \
--managed
```
- This creates:
```
VPC
EKS control plane
Worker nodes
```
- Check nodes
```
kubectl get nodes
```

# Kubernetes Deployment Files
```
mkdir k8s
cd k8s
```
## In Kubernetes we create
- deployments
- services
  
## MongoDB Deployment
- nano mongodb-deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:6
        ports:
        - containerPort: 27017
```
- Service
```
nano mongodb-service.yaml
```
```
apiVersion: v1
kind: Service
metadata:
  name: mongodb
spec:
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017
```
## Backend Deployment
```
backend-deployment.yaml
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: vignesh0777/mean-backend:latest
        ports:
        - containerPort: 8080
        env:
        - name: MONGO_URL
          value: mongodb://mongodb:27017/dd_db
```

- Service
```
nano backend-service.yaml
```
```
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080
```


## Frontend Deployment
```
nano frontend-deployment.yaml
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: vignesh0777/mean-frontend:latest
        ports:
        - containerPort: 80
```
- Service
```
 nano frontend-service.yaml
```
```
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```
## Deploy to Kubernetes
- Run on Jenkins EC2
```
kubectl apply -f mongodb-deployment.yaml
kubectl apply -f mongodb-service.yaml

kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml

kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml
```
- Check
```
kubectl get pods
kubectl get svc
```

## Enable Pod Auto Scaling
```
kubectl autoscale deployment backend --cpu-percent=50 --min=2 --max=6
```
- Check
```
kubectl get hpa
```
Now pods automatically scale
## Jenkins Pipeline (Auto Deploy)
- Create Jenkins Pipeline Job
Pipeline script

```
pipeline {
 agent any

 stages {

 stage('Clone Repo'){
 steps{
 git branch: 'main', url: 'https://github.com/vigneshreddy3129/mean-devops-assignment.git'
 }
 }

 stage('Build Images'){
 steps{
 sh 'docker build -t vignesh0777/mean-backend ./backend'
 sh 'docker build -t vignesh0777/mean-frontend ./frontend'
 }
 }
 stage('Docker Login'){
 steps{
  sh 'docker login -u vignesh0777 -p Vignesh123'
 }
}

 stage('Push Images'){
 steps{
 sh 'docker push vignesh0777/mean-backend'
 sh 'docker push vignesh0777/mean-frontend'
 }
 }

 stage('Deploy to EKS'){
 steps{
 sh 'kubectl apply -f k8s/'
 sh 'kubectl rollout restart deployment backend'
 sh 'kubectl rollout restart deployment frontend'
 }
 }

 }
}
```
## GitHub Webhook
- GitHub repo
Settings
Webhooks
```
- Add
http://JENKINS_IP:8080/github-webhook/
```
# Now whenever developer pushes code:
---
GitHub → Jenkins → Build Image → Push → Deploy to EKS → Pods Updated
---

# What Happens Now (Automation)
- When developer pushes code:
```
GitHub Push
↓
Jenkins Pipeline
↓
Docker Images Rebuilt
↓
Images pushed to DockerHub
↓
EKS pulls new images
↓
Pods restart automatically
↓
Traffic served via LoadBalancer
↓
HPA scales pods
↓
Nodegroup scales servers
```


