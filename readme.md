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
- Docker (latest)  
- Minikube (latest)  
- kubectl (latest)  
- Docker Hub account


🔧 Step-1: Start Minikube  
- Start cluster:  
```
minikube start
```
- Enable Metrics Server (required for HPA autoscaling):  
```
minikube addons enable metrics-server
```
![Docker push](images/Screenshot%20(20).png)

🐳 Step-2: Build & Push Docker Image  
- From the project root (where Dockerfile exists):  
```
docker build -t flask-app:1.0 .
docker tag flask-app:1.0 <YOUR_DOCKERHUB_USERNAME>/flask-app:1.0
docker login
docker push <YOUR_DOCKERHUB_USERNAME>/flask-app:1.0
```

🔐 Step-3: Deploy MongoDB Database  
- Apply Persistent Volume:  
```
kubectl apply -f k8s/mongo-pv.yml
```
- Apply Secret (DB auth):  
```
kubectl apply -f k8s/mongo-secret.yml
```
- Deploy StatefulSet:  
```
kubectl apply -f k8s/mongo-statefulset.yml
```
- Deploy MongoDB Service:  
```
kubectl apply -f k8s/mongo-service.yml
```
- Verify:  
```
kubectl get pods
kubectl get pv
kubectl get pvc
kubectl get svc
```


📸 Screenshot 3: mongo-0 running + PV/PVC Bound  
![Mongo StatefulSet running](images/Screenshot%20(21).png)

🌐 Step-4: Deploy Flask Application  
```
kubectl apply -f k8s/flask-deployment.yml
kubectl apply -f k8s/flask-service.yml
```
- Check:  
```
kubectl get pods
kubectl get svc
```

🌍 Step-5: Access the Flask App  
- Port-forward and open:  
```
kubectl port-forward deployment/flask-app 5000:5000
```
Visit: http://localhost:5000/
![HPA status](images/Screenshot%20(24).png)

📈 Step-6: Enable Auto-Scaling (HPA)  
- Apply HPA and verify:  
```
kubectl apply -f k8s/flask-hpa.yml
kubectl get hpa
```


📸 Screenshot 6: HPA showing min=2, max=5  
![HPA status](images/Screenshot%20(22).png)

🧪 Load Testing (simulate high CPU)  
- Enter a Flask pod:  
```
kubectl exec -it <flask-pod-name> -- bash
apt update && apt install -y stress
stress --cpu 2 --timeout 120s
```
- Watch scaling:  
```
kubectl get pods -w
kubectl get hpa -w
```


You should see new pods appearing automatically ➜ scaling from 2 → 4 → 5

📸 Screenshot 7: New Flask pods created by HPA  
![Scaled Flask pods](images/Screenshot%20(23).png)


📌 DNS Resolution in Kubernetes (Inter-Pod Communication)

Kubernetes uses CoreDNS to automatically create DNS entries for Services and Pods.
This allows applications inside the cluster to communicate using service names instead of IP addresses.

In this project, MongoDB runs as a StatefulSet behind a Headless Service named mongo.
Because StatefulSets assign a stable hostname to each pod, MongoDB pod receives a DNS name:

mongo-0.mongo
```
- Flask connects via:  
```
mongodb://mongo-user:password@mongo-0.mongo:27017/flask_db?authSource=admin


➡️ CoreDNS resolves mongo-0.mongo → correct MongoDB Pod IP
➡️ Works even if Pod IP changes or Pod restarts
➡️ MongoDB remains internal only, not exposed outside the cluster

DNS ensures reliable and secure inter-pod communication without hard-coded IPs.
```
- Works across restarts; MongoDB stays internal-only.

📌 Resource Requests & Limits in Kubernetes  
- Manifest snippet:  
```yaml
resources:
  requests:
    cpu: "0.2"
    memory: "250Mi"
  limits:
    cpu: "0.5"
    memory: "500Mi"
```
- Requests = minimum for scheduling (≥0.2 CPU, 250Mi RAM).  
- Limits = max before throttling/eviction (0.5 CPU, 500Mi RAM).  

Benefits:  
| Benefit         | Explanation                                     |
|-----------------|-------------------------------------------------|
| Stability       | Stops one app from hogging all resources        |
| Predictability  | Keeps performance consistent under load         |
| Required for HPA| CPU % drives autoscaling decisions              |

📌 Design Choices & Alternatives  
| Design Choice              | Reason                                      | Alternatives & Why Not Used                     |
|----------------------------|---------------------------------------------|-------------------------------------------------|
| Flask as a Deployment      | Stateless API, needs scaling/rollouts       | StatefulSet unnecessary (no stable identity)    |
| MongoDB as StatefulSet     | Stable hostname + persistent storage        | Deployment would break DB identity on restarts  |
| Headless Service for Mongo | Pod-specific DNS (`mongo-0.mongo`)          | ClusterIP alone lacks per-pod DNS               |
| NodePort for Flask Service | Local access from Minikube                  | LoadBalancer not available locally              |
| hostPath PV for Mongo      | Best fit for local Minikube dev             | Cloud volumes unavailable locally               |
| Secret for credentials     | Secure MongoDB auth                         | Hard-coding creds is insecure                   |
| HPA on CPU metrics         | Metrics Server supports CPU out-of-box      | Custom metrics add setup complexity             |

🍪 Testing Scenarios & Results  
- Database functionality  
  - POST data:  
    ```
    POST http://localhost:5000/data
    Body: {"name": "Test", "role": "Developer"}
    ```
    Response: `{"status":"Data inserted"}`
  - GET data:  
    ```
    GET http://localhost:5000/data
    ```
    ✔ CRUD confirmed; ✔ DNS via `mongo-0.mongo`; ✔ Auth worked; ✔ Data persisted across restarts.  
  - 📸 Add screenshot of successful CRUD run here (if available).
![Scaled Flask pods](images/Screenshot%20(25).png)


- HPA autoscaling (high CPU)  
  - Inside a Flask pod:  
    ```
    kubectl exec -it <flask-pod-name> -- bash
    apt update && apt install -y stress
    stress --cpu 2 --timeout 120s
    ```
  - Monitor scaling:  
    ```
    kubectl get hpa -w
    kubectl get pods -w
    ```
  - Result: scaled from 2 → 4 pods automatically 🎯  
    - Before load: replicas = 2  
    - After load: pods increased (see HPA screenshot above)
![Scaled Flask pods](images/Screenshot%20(23).png)


Issues Encountered & Fixes

Autoscaling was not working initially
    • I forgot to enable the Metrics Server, so CPU usage was not being monitored
    • After enabling it using minikube addons enable metrics-server and restarting it, HPA started scaling correctly

MongoDB pod stuck in Pending state
    • Old PV/PVC from a previous configuration were still bound
    • I deleted the old StatefulSet, PV, and PVC, then redeployed MongoDB → storage bound successfully

Unable to access Flask app in browser at first
    • I was trying to use localhost instead of the Minikube node IP
    • Using minikube ip with NodePort worked correctly

Flask initially failed to connect to MongoDB
    • StatefulSet takes a little time to initialize MongoDB on first startup
    • Waiting until the mongo-0 pod status became Running resolved it
