# EKS-Jenkins-ECR-Kubernetes-CI-CD-Pipeline

🚀 STEP 1 — Create EKS Cluster
📍 Done in AWS Console
Go to EKS → Create Cluster
Create Node Group
Min: 2
Desired: 2
Max: 4
Cluster ready.
🚀 STEP 2 — Launch Jenkins Server (EC2)
📍 AWS Console → EC2 → Launch instance
Install on Jenkins VM:
Bash
Copy code
sudo yum update -y
sudo yum install docker -y
sudo systemctl start docker
sudo usermod -aG docker ec2-user

sudo yum install java-17 -y
sudo yum install jenkins -y
sudo systemctl start jenkins
Install:
AWS CLI
kubectl
Connect Jenkins to EKS:
Bash
Copy code
aws configure
aws eks update-kubeconfig --region ap-south-1 --name my-cluster
kubectl get nodes
If nodes appear → Jenkins can deploy.
🚀 STEP 3 — Create ECR Repository
📍 AWS Console → ECR → Create repo
Example name: myapp
Copy ECR URL:
Copy code

123456789.dkr.ecr.ap-south-1.amazonaws.com/myapp
🚀 STEP 4 — Create Project Structure (On Your Local Machine)
Create this structure:
Copy code

myapp/
├── app.js
├── package.json
├── Dockerfile
├── Jenkinsfile   👈 IMPORTANT
└── k8s/
├── deployment.yaml
├── service.yaml
├── ingress.yaml
└── hpa.yaml
Push this to GitHub.
🚀 STEP 5 — Dockerfile
Inside project root:
Dockerfile
Copy code
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "app.js"]
🚀 STEP 6 — Kubernetes Files (Inside k8s Folder)
deployment.yaml
YAML
Copy code
apiVersion: apps/v1
kind: Deployment
metadata:
name: myapp
spec:
replicas: 2
selector:
matchLabels:
app: myapp
template:
metadata:
labels:
app: myapp
spec:
containers:

name: myapp
image: IMAGE_PLACEHOLDER
ports:

containerPort: 3000
resources:
requests:
cpu: "200m"
limits:
cpu: "500m"
service.yaml
YAML
Copy code
apiVersion: v1
kind: Service
metadata:
name: myapp-service
spec:
selector:
app: myapp
ports:

port: 80
targetPort: 3000
type: ClusterIP
ingress.yaml
YAML
Copy code
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
name: myapp-ingress
spec:
rules:


http:
paths:

path: /
pathType: Prefix
backend:
service:
name: myapp-service
port:
number: 80
hpa.yaml
YAML
Copy code
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
name: myapp-hpa
spec:
scaleTargetRef:
apiVersion: apps/v1
kind: Deployment
name: myapp
minReplicas: 2
maxReplicas: 6
metrics:

type: Resource
resource:
name: cpu
target:
type: Utilization
averageUtilization: 70
🚀 STEP 7 — Create Jenkinsfile (IMPORTANT)
📍 This file goes in your GitHub repo root.
Groovy
Copy code
pipeline {
agent any

environment {
AWS_REGION = "ap-south-1"
ECR_URL = "123456789.dkr.ecr.ap-south-1.amazonaws.com/myapp"
}

stages {

stage('Clone') {
steps {
git 'https://github.com/yourrepo.git'
}
}

stage('Build Docker') {
steps {
sh 'docker build -t myapp:$BUILD_NUMBER .'
}
}

stage('Login to ECR') {
steps {
sh '''
aws ecr get-login-password --region $AWS_REGION |
docker login --username AWS --password-stdin $ECR_URL
'''
}
}

stage('Push to ECR') {
steps {
sh '''
docker tag myapp:$BUILD_NUMBER $ECR_URL:$BUILD_NUMBER
docker push $ECR_URL:$BUILD_NUMBER
'''
}
}

stage('Deploy to EKS') {
steps {
sh '''
sed -i "s|IMAGE_PLACEHOLDER|$ECR_URL:$BUILD_NUMBER|g" k8s/deployment.yaml
kubectl apply -f k8s/
'''
}
}

}
}
🚀 STEP 8 — Create Jenkins Pipeline Job
📍 Jenkins UI
New Item
Choose Pipeline
In Pipeline section → choose:
Copy code

Pipeline script from SCM
Select Git
Paste GitHub repo URL
Save
Now Jenkins will automatically read Jenkinsfile.
🔥 WHAT HAPPENS WHEN YOU CLICK BUILD
1️⃣ Jenkins clones repo
2️⃣ Builds Docker image
3️⃣ Pushes to ECR
4️⃣ Replaces image in deployment.yaml
5️⃣ Applies k8s folder
6️⃣ Kubernetes rolling update starts
🔥 SCALING BEHAVIOR
Traffic increases:
CPU > 70% → HPA increases pods
Nodes full → Cluster Autoscaler adds new VM
Fully automatic.
🎯 WHERE EVERYTHING RUNS
Task
Runs Where
Create cluster
AWS Console
Write code
Local machine
Push to GitHub
Local machine
Build Docker
Jenkins server
Push to ECR
Jenkins server
kubectl apply
Jenkins server
Scaling
Kubernetes
🎤 FINAL INTERVIEW ANSWER
“We store Dockerfile, Kubernetes manifests, and Jenkinsfile in GitHub. Jenkins pipeline builds and pushes the image to ECR, updates the Deployment, and applies Kubernetes manifests. HPA scales pods based on CPU, and Cluster Autoscaler provisions additional nodes if required.”
