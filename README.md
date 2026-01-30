# 3-Tier Application Deployment using Docker + Kubernetes (AWS EKS)
This repository documents the setup and deployment of a 3-tier web application using:
* Docker (build + push images to Docker Hub)  
* Kubernetes on AWS EKS (deploy backend + frontend using YAML)  
* AWS Console (EC2 instances, EKS Cluster + Node Group created from AWS Account UI)  


## Architecture / Flow
RDS (MariaDB)  -->  Backend (Spring Boot :8080)  -->  Frontend (Apache :80)
```
Build → Push → Deploy flow
```
* For Backend —>> Update configs → Build Docker image → Push to Docker Hub → create pod.yaml file (keep image: as < DockerHub Username>/backend:v1)→ create svc.yaml file and expose it to 8080 port for target port and ports.  
* For Frontend —>> Update configs → Build Docker image → Push to Docker Hub → create deploy.yaml file expose it to 80 port for target port and ports.  
* Kubernetes pulls the Docker Hub images (image name is written inside YAML)  
* Expose both services using LoadBalancer  
    * Backend Service: 8080  
    * Frontend Service: 80

⸻

## Tech Stack
* Frontend: Node.js, npm, Apache  
* Backend: Java 17, Maven, Spring Boot  
* Database: MariaDB (AWS RDS)  
* Containerization: Docker  
* Orchestration: Kubernetes (AWS EKS) 

⸻

1) Database Setup (AWS RDS – MariaDB)
	1.	Go to Amazon RDS → Create database
	2.	Choose:
	•	Engine: MariaDB
	•	Template: Free tier
	•	Set username & password
	•	Storage: gp2
	3.	Security Group
	•	Allow inbound on 3306 from your EC2 / EKS node security group
	4.	Create DB and note RDS Endpoint

Use these variables:
```
DB_HOST = <rds-endpoint>
DB_USER = <db-username>
DB_PASS = <db-password>
DB_NAME = student_db
```


⸻

2) Create Schema / Table (from EC2)
```
sudo apt update -y
sudo apt install mysql-client -y
mysql -h <RDS-ENDPOINT> -u <USERNAME> -p
```
```
CREATE DATABASE student_db;
USE student_db;

```
```
CREATE TABLE `students` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  `email` varchar(255) DEFAULT NULL,
  `course` varchar(255) DEFAULT NULL,
  `student_class` varchar(255) DEFAULT NULL,
  `percentage` double DEFAULT NULL,
  `branch` varchar(255) DEFAULT NULL,
  `mobile_number` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
);

EXIT;
```
✅ Immediately after RDS is ready → Create EKS Cluster + Node Group (AWS Console).

⸻

3) Create EKS Cluster + Node Group (AWS Console – Step by Step)

Cluster and nodes are created from AWS Account UI (Console), not terminal.

Step A — Create EKS Cluster

	1.	Open AWS Console → EKS
	2.	Click Clusters → Add cluster → Create Cluster
	3.	Fill:
	•	Name: cluster (or your name)
	•	Kubernetes version: latest stable
	4.	Cluster Service Role
	•	Create/select IAM Role with: AmazonEKSClusterPolicy
	5.	Networking 
	•	Choose security group
	6.	Click Create

Wait until Cluster status becomes Active.

⸻

Step B — Create Node Group

	1.	In EKS → open your cluster
	2.	Go to Compute → Add Node Group
	3.	Fill:
	•	Node group name: nodegroup-1 (example)
	4.	Node IAM role must include:
	•	AmazonEKSWorkerNodePolicy
	•	AmazonEKSWorkerNodeMinimalPolicy
	•	AmazonEC2ContainerRegistryReadOnly
	•	AmazonEKS_CNI_Policy
	5.	Compute settings
	•	Instance type: c7i-flex.large
	•	Desired size: 2
	•	Min: 1
	•	Max: 3
	6.	Choose subnets → Create

Wait until Node Group status becomes Active.

⸻

4) One-Time EC2 Setup (kubectl + AWS CLI + kubeconfig)

These steps are done once, after DB and EKS are ready.

A) Install Docker (EC2)
```
sudo apt install docker.io -y
```
B) Install kubectl (EC2) --> From Kubernates Indtall Git
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```
C) Install AWS CLI (EC2)
```
snap install aws-cli --classic
aws --version
```
D) Configure AWS + Connect kubectl to EKS
```
aws configure
aws eks update-kubeconfig --name <ClusterName>
kubectl get nodes
```
✅ If nodes appear, EC2 is connected to the EKS cluster.

⸻

5) Clone the Repo (EC2)
```
git clone https://github.com/Rohit-1920/EasyCRUD.git
```

⸻

6) Backend (Docker Build + Push to Docker Hub)

Update DB properties
```
cd EasyCRUD/backend/
cp src/main/resources/application.properties .
nano application.properties
```
Update values in the application.properties file:
```
server.port=8080
spring.datasource.url=jdbc:mariadb://<DB_HOST>:3306/<DB_NAME>
spring.datasource.username=<DB_USER>
spring.datasource.password=<DB_PASS>
```

Build + Run locally (optional test)
```
nano Dockerfile
docker build -t backend:v1 .
docker images
```
Use a predefined Maven + Java image in Dockerfile:
```
FROM maven:3.8.5-openjdk-17
COPY . /opt/
WORKDIR /opt
RUN rm -rf src/main/resources/application.properties
RUN cp -r application.properties src/main/resources/
RUN mvn clean package
WORKDIR target/
CMD ["java","-jar","student-registration-backend-0.0.1-SNAPSHOT.jar"]
```
Use the JAR name from your pom.xml (artifactId).

Tag + Push to Docker Hub
```
docker login -u<dockerhub-username>
docker tag backend:v1 <dockerhub-username>/backend:v1
docker push <dockerhub-username>/backend:v1
 ```
OR —Optional—
```
docker login -u<dockerhub-username>
docker build -t <username>/backend:v1 .
docker push <dockerhub-username>/backend:v1
```
⸻

7) Frontend (Docker Build + Push to Docker Hub)
```
cd ../frontend/
nano .env
```
Set backend URL in the .env file:
```
REACT_APP_BACKEND_URL=http://<PUBLIC ID OF EC@>:8080
```
Build + Push:
```
nano Dockerfile
docker build -t frontend:v1 .
docker tag frontend:v1 <dockerhub-username>/frontend:v1
docker push <dockerhub-username>/frontend:v1
```
Use a predefined apache image in Dockerfile:
```
FROM node:25-alpine
WORKDIR /opt
COPY . /opt/
RUN npm install
RUN npm run build
RUN apk update
RUN apk add apache2
RUN cp -rf dist/* /var/www/localhost/htdocs/
EXPOSE 80
ENTRYPOINT ["httpd","-D","FOREGROUND"]
```

⸻

8) Kubernetes YAML Files (Backend + Frontend)

You created these YAMLs:

	•	Backend: backend-pod.yml and backend-svc.yml
	•	Frontend: deploy.yml and frontend-svc.yml
	•	Both services are LoadBalancer
	•	Backend port 8080
	•	Frontend port 80
	•	Images are pulled from Docker Hub

⸻

Backend Pod (backend-pod.yml)
```
apiVersion: v1
kind: Pod
metadata:
  name: backend-pod
  labels:
    app: backend
    env: dev
spec:
  containers:
    - name: backend
      image: mimanasi/backend:v1
      ports:
        - containerPort: 8080
```

Backend Service (backend-svc.yml) — LoadBalancer (8080)
```
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  labels:
    app: backend
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
    - port: 8080
      targetPort: 8080

```

⸻

Frontend Deployment (deploy.yml)
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  labels:
    app: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        env: dev
    spec:
      containers:
        - name: frontend
          image: mimanasi/frontend:v1
          ports:
            - containerPort: 80
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
```

Frontend Service (frontend-svc.yml) — LoadBalancer (80)
```
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  labels:
    app: frontend
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
```

⸻

9) Apply YAML Files to EKS

Backend
```
cd EasyCRUD/backend/
kubectl apply -f backend-pod.yml
kubectl apply -f backend-svc.yml
kubectl get svc
```
Frontend
```
cd EasyCRUD/frontend/
kubectl apply -f deploy.yml
kubectl apply -f frontend-svc.yml
kubectl get svc
```

⸻

10) Access the Application

After LoadBalancers are ready:

kubectl get svc

	•	Backend: http://<backend-lb-dns>:8080
	•	Frontend: http://<frontend-lb-dns>:80

⸻

# Commands Used (from EC2 terminal history)

Below is the same flow you executed (cleaned and grouped):
```
# Install dependencies

sudo apt install mysql-client -y
sudo apt install docker.io -y


# Clone repo

git clone https://github.com/Rohit-1920/EasyCRUD.git


# Connect to DB

mysql -h <RDS-ENDPOINT> -u admin -p

# Backend build + push
cd EasyCRUD/backend/
cp src/main/resources/application.properties .
nano application.properties
nano Dockerfile
docker build -t backend:v1 .
docker run -d -p 8080:8080 backend:v1
docker ps
docker login -u<dockerhub-username>
docker tag backend:v1 <dockerhub-username>/backend:v1
docker push <dockerhub-username>/backend:v1

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client

# AWS CLI + kubeconfig
snap install aws-cli --classic
aws configure
aws eks update-kubeconfig --name <ClusterName>
kubectl get nodes

# Backend YAML
nano backend-pod.yml
nano backend-svc.yml
kubectl apply -f backend-pod.yml
kubectl apply -f backend-svc.yml
kubectl get svc

# Frontend build + push
cd ../frontend/
nano .env
nano Dockerfile
docker build -t frontend:v1 .
docker tag frontend:v1 <dockerhub-username>/frontend:v1
docker push <dockerhub-username>/frontend:v1

# Frontend YAML
nano deploy.yml
nano frontend-svc.yml
kubectl apply -f deploy.yml
kubectl apply -f frontend-svc.yml
kubectl get svc
```

⸻

Debug Commands
```
kubectl get pods
kubectl get deploy
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl get events
kubectl delete -f <file.yaml>
```

⸻

Final Outcome
	•	RDS MariaDB running securely
	•	Backend running on EKS and exposed via LoadBalancer on 8080
	•	Frontend running on EKS and exposed via LoadBalancer on 80
	•	Both workloads pull Docker Hub images directly from YAML

⸻
