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

# 🚀 STEP 1 — Create EKS Cluster

📍 AWS Console → EKS → Create Cluster

1. Create Cluster
2. Create Node Group  
   - Min: 2  
   - Desired: 2  
   - Max: 4  

Cluster ready ✅

---

# 🚀 STEP 2 — Launch Jenkins Server (EC2)

📍 AWS Console → EC2 → Launch Instance

## Install Required Packages

```bash
sudo yum update -y
sudo yum install docker -y
sudo systemctl start docker
sudo usermod -aG docker ec2-user

sudo yum install java-17 -y
sudo yum install jenkins -y
sudo systemctl start jenkins
```

## Install Tools
- AWS CLI
- kubectl

## Connect Jenkins to EKS

```bash
aws configure
aws eks update-kubeconfig --region ap-south-1 --name my-cluster
kubectl get nodes
```

If nodes appear → Jenkins can deploy ✅

---

# 🚀 STEP 3 — Create ECR Repository

📍 AWS Console → ECR → Create Repository

Example repository name:
```
myapp
```

Example ECR URL:
```
123456789.dkr.ecr.ap-south-1.amazonaws.com/myapp
```

---

# 🚀 STEP 4 — Project Structure

```
myapp/
├── app.js
├── package.json
├── Dockerfile
├── Jenkinsfile
└── k8s/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    └── hpa.yaml
```

Push this project to GitHub.

---

# 🚀 STEP 5 — Dockerfile

```dockerfile
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "app.js"]
```

---

# 🚀 STEP 6 — Kubernetes Manifests

## deployment.yaml

```yaml
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
      - name: myapp
        image: IMAGE_PLACEHOLDER
        ports:
        - containerPort: 3000
        resources:
          requests:
            cpu: "200m"
          limits:
            cpu: "500m"
```

---

## service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP
```

---

## ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

---

## hpa.yaml

```yaml
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
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

# 🚀 STEP 7 — Jenkinsfile

Place this file in the project root.

```groovy
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
        aws ecr get-login-password --region $AWS_REGION \
        | docker login --username AWS --password-stdin $ECR_URL
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
```

---

# 🚀 STEP 8 — Create Jenkins Pipeline Job

📍 Jenkins UI

1. Click **New Item**
2. Select **Pipeline**
3. Choose **Pipeline script from SCM**
4. Select **Git**
5. Paste GitHub repository URL
6. Save

Jenkins automatically reads the `Jenkinsfile`.

---

# 🔥 What Happens When You Click Build?

1. Jenkins clones repository  
2. Builds Docker image  
3. Pushes image to ECR  
4. Updates image in deployment.yaml  
5. Applies Kubernetes manifests  
6. Rolling update starts  

---

# 🔥 Scaling Behavior

- CPU > 70% → HPA increases pods
- If nodes are full → Cluster Autoscaler adds new EC2 nodes
- Fully automatic scaling

---

# 🎯 Where Everything Runs

| Task | Runs On |
|------|---------|
| Create Cluster | AWS Console |
| Write Code | Local Machine |
| Push to GitHub | Local Machine |
| Docker Build | Jenkins Server |
| Push to ECR | Jenkins Server |
| kubectl apply | Jenkins Server |
| Auto Scaling | Kubernetes |

---

# 🎤 Interview Summary

We store Dockerfile, Kubernetes manifests, and Jenkinsfile in GitHub.  
Jenkins builds the Docker image, pushes it to ECR, updates the Kubernetes deployment, and applies manifests to EKS.  
HPA scales pods based on CPU utilization, and Cluster Autoscaler provisions additional nodes when required.

---

✅ End of Project Documentation
