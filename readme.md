🚀 Project Overview

This project deploys a Flask application and a MongoDB database on a Minikube Kubernetes cluster.
It demonstrates:
✔ Deployments, StatefulSets, Services
✔ Persistent Volume for MongoDB
✔ Authentication via Kubernetes Secrets
✔ Horizontal Pod Autoscaling based on CPU
✔ DNS-based communication between services

User ➜ NodePort Service ➜ Flask Deployment (autoscaling) ➜ DNS ➜ MongoDB StatefulSet ➜ PV/PVC

🛠️ Prerequisites

Ensure you have the following installed:

Tool	Version
Docker	Latest
Minikube	Latest
kubectl	Latest
Docker Hub Account	Required


🔧 Step-1: Start Minikube
minikube start


Enable Metrics Server (required for HPA autoscaling):

minikube addons enable metrics-server

![Docker push](images/Screenshot%20(20).png)

🐳 Step-2: Build & Push Docker Image

Inside the Flask project folder (where Dockerfile exists):

docker build -t flask-app:1.0 .
docker tag flask-app:1.0 <YOUR_DOCKERHUB_USERNAME>/flask-app:1.0
docker login
docker push <YOUR_DOCKERHUB_USERNAME>/flask-app:1.0


📸 Screenshot 2: Successful docker push  

🔐 Step-3: Deploy MongoDB Database

Apply Persistent Volume:

kubectl apply -f k8s/mongo-pv.yml


Apply Secret (DB auth):

kubectl apply -f k8s/mongo-secret.yml


Deploy StatefulSet:

kubectl apply -f k8s/mongo-statefulset.yml


Deploy MongoDB Service:

kubectl apply -f k8s/mongo-service.yml


Verify:

kubectl get pods
kubectl get pv
kubectl get pvc
kubectl get svc


📸 Screenshot 3: mongo-0 running + PV/PVC Bound  
![Mongo StatefulSet running](images/Screenshot%20(21).png)

🌐 Step-4: Deploy Flask Application
kubectl apply -f k8s/flask-deployment.yml
kubectl apply -f k8s/flask-service.yml


Check:

kubectl get pods
kubectl get svc

🌍 Step-5: Access the Flask App

Open in browser:

kubectl port-forward deployment/flask-app 5000:5000
http://localhost:5000/

![HPA status](images/Screenshot%20(24).png)

📈 Step-6: Enable Auto-Scaling (HPA)

Apply HPA:

kubectl apply -f k8s/flask-hpa.yml
kubectl get hpa


📸 Screenshot 6: HPA showing min=2, max=5  
![HPA status](images/Screenshot%20(22).png)

🧪 Load Testing (simulate high CPU)

Enter a Flask pod:

kubectl exec -it <flask-pod-name> -- bash
apt update && apt install -y stress
stress --cpu 2 --timeout 120s


Watch scaling:

kubectl get pods -w
kubectl get hpa -w


You should see new pods appearing automatically ➜ scaling from 2 → 4 → 5

📸 Screenshot 7: New Flask pods created by HPA  
![Scaled Flask pods](images/Screenshot%20(23).png)

