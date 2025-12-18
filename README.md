# Flask + MongoDB on Kubernetes (Minikube)

This project implements the FarAlpha intern assignment: a Python Flask API backed by MongoDB, deployed on a local Kubernetes cluster (Minikube) with authentication, persistent storage, Services, Horizontal Pod Autoscaler, DNS-based service discovery, and resource requests/limits.

---

## 1. Application overview

- `GET /`  
  Returns: `Welcome to the Flask app! The current time is: <Date and Time>`.

- `GET /data`  
  Returns all documents from MongoDB (excluding `_id`).

- `POST /data`  
  Inserts a JSON document into MongoDB.

The Flask app connects to MongoDB using username/password from environment variables, which are sourced from a Kubernetes Secret.

---

## 2. Local development (Part 1)

> Part 1 is not required for submission but is included here for completeness. 

### 2.1 Setup

mkdir flask-mongodb-app
cd flask-mongodb-app

python -m venv venv

Windows
venv\Scripts\activate

Linux/macOS
source venv/bin/activate
pip install -r requirements.txt

text

**Virtual environment benefits (cookie point):**

- Isolates dependencies per project so different apps can use different versions without conflicts.  
- Avoids polluting global Python/site‑packages and makes deployments reproducible.

### 2.2 Run MongoDB locally with Docker

docker pull mongo:latest
docker run -d -p 27017:27017 --name mongodb mongo:latest

text

Create `.env`:

MONGODB_URI=mongodb://localhost:27017/

text

Run Flask (Windows example):

set FLASK_APP=app.py
set FLASK_ENV=development
flask run

text

Test:

curl http://localhost:5000/
curl -X POST -H "Content-Type: application/json" -d "{"sampleKey":"sampleValue"}" http://localhost:5000/data
curl http://localhost:5000/data

text

---

## 3. Flask application (Kubernetes-ready)

`app.py`:

from flask import Flask, request, jsonify
from pymongo import MongoClient
from datetime import datetime
import os

app = Flask(name)

MONGO_USER = os.environ.get("MONGO_USER", "root")
MONGO_PASSWORD = os.environ.get("MONGO_PASSWORD", "examplepassword")
MONGO_HOST = os.environ.get("MONGO_HOST", "mongo")
MONGO_PORT = os.environ.get("MONGO_PORT", "27017")
MONGO_DB = os.environ.get("MONGO_DB", "flask_db")
MONGO_AUTH_DB = os.environ.get("MONGO_AUTH_DB", "admin")

MONGODB_URI = (
f"mongodb://{MONGO_USER}:{MONGO_PASSWORD}@{MONGO_HOST}:{MONGO_PORT}/"
f"{MONGO_DB}?authSource={MONGO_AUTH_DB}"
)

client = MongoClient(MONGODB_URI)
db = client[MONGO_DB]
collection = db.data

@app.route("/")
def index():
return f"Welcome to the Flask app! The current time is: {datetime.now()}"

@app.route("/data", methods=["GET", "POST"])
def data():
if request.method == "POST":
data = request.get_json()
collection.insert_one(data)
return jsonify({"status": "Data inserted"}), 201
docs = list(collection.find({}, {"_id": 0}))
return jsonify(docs), 200

if name == "main":
app.run(host="0.0.0.0", port=5000)

text

---

## 4. Dockerfile and image

`Dockerfile`:

FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

ENV FLASK_APP=app.py
ENV FLASK_ENV=production

EXPOSE 5000

CMD ["flask", "run", "--host=0.0.0.0", "--port=5000"]

text

### Build and push

docker build -t tr3n/flask-mongodb-app:latest .
docker login
docker push tr3n/flask-mongodb-app:latest

text

This satisfies the “Dockerfile + build/push instructions” requirement. [file:223]

---

## 5. Kubernetes manifests

### 5.1 MongoDB Secret

`mongo-secret.yaml`:

apiVersion: v1
kind: Secret
metadata:
name: mongo-secret
type: Opaque
stringData:
mongo-root-username: root
mongo-root-password: examplepassword

text

### 5.2 Storage: PV and PVC

`mongo-pv.yaml`:

apiVersion: v1
kind: PersistentVolume
metadata:
name: mongo-pv
spec:
capacity:
storage: 1Gi
accessModes:
- ReadWriteOnce
hostPath:
path: /data/mongo

text

`mongo-pvc.yaml`:

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
name: mongo-pvc
spec:
accessModes:
- ReadWriteOnce
resources:
requests:
storage: 1Gi

text

### 5.3 Mongo Service and StatefulSet

`mongo-service.yaml`:

apiVersion: v1
kind: Service
metadata:
name: mongo
labels:
app: mongo
spec:
clusterIP: None
ports:
- port: 27017
targetPort: 27017
selector:
app: mongo

text

`mongo-statefulset.yaml`:

apiVersion: apps/v1
kind: StatefulSet
metadata:
name: mongo
spec:
serviceName: "mongo"
replicas: 1
selector:
matchLabels:
app: mongo
template:
metadata:
labels:
app: mongo
spec:
containers:
- name: mongo
image: mongo:6.0
ports:
- containerPort: 27017
env:
- name: MONGO_INITDB_ROOT_USERNAME
valueFrom:
secretKeyRef:
name: mongo-secret
key: mongo-root-username
- name: MONGO_INITDB_ROOT_PASSWORD
valueFrom:
secretKeyRef:
name: mongo-secret
key: mongo-root-password
- name: MONGO_INITDB_DATABASE
value: flask_db
volumeMounts:
- name: mongo-storage
mountPath: /data/db
resources:
requests:
cpu: "0.2"
memory: "250Mi"
limits:
cpu: "0.5"
memory: "500Mi"
volumeClaimTemplates:
- metadata:
name: mongo-storage
spec:
accessModes: ["ReadWriteOnce"]
resources:
requests:
storage: 1Gi

text

### 5.4 Flask Deployment, Service, HPA

`flask-deployment.yaml`:

apiVersion: apps/v1
kind: Deployment
metadata:
name: flask-app
spec:
replicas: 2
selector:
matchLabels:
app: flask-app
template:
metadata:
labels:
app: flask-app
spec:
containers:
- name: flask-app
image: tr3n/flask-mongodb-app:latest
imagePullPolicy: Always
ports:
- containerPort: 5000
env:
- name: MONGO_USER
valueFrom:
secretKeyRef:
name: mongo-secret
key: mongo-root-username
- name: MONGO_PASSWORD
valueFrom:
secretKeyRef:
name: mongo-secret
key: mongo-root-password
- name: MONGO_HOST
value: "mongo"
- name: MONGO_PORT
value: "27017"
- name: MONGO_DB
value: "flask_db"
- name: MONGO_AUTH_DB
value: "admin"
resources:
requests:
cpu: "0.2"
memory: "250Mi"
limits:
cpu: "0.5"
memory: "500Mi"

text

`flask-service.yaml`:

apiVersion: v1
kind: Service
metadata:
name: flask-service
spec:
type: NodePort
selector:
app: flask-app
ports:
- port: 5000
targetPort: 5000
nodePort: 30007

text

`flask-hpa.yaml`:

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
name: flask-app-hpa
spec:
scaleTargetRef:
apiVersion: apps/v1
kind: Deployment
name: flask-app
minReplicas: 2
maxReplicas: 5
metrics:
- type: Resource
resource:
name: cpu
target:
type: Utilization
averageUtilization: 70

text

---

## 6. Deployment steps on Minikube

Start Minikube:

minikube start --memory=4096 --cpus=2

text

Apply MongoDB resources:

kubectl apply -f mongo-secret.yaml
kubectl apply -f mongo-pv.yaml
kubectl apply -f mongo-pvc.yaml
kubectl apply -f mongo-service.yaml
kubectl apply -f mongo-statefulset.yaml
kubectl get pods

text

Apply Flask resources:

kubectl apply -f flask-deployment.yaml
kubectl apply -f flask-service.yaml
kubectl apply -f flask-hpa.yaml

kubectl get pods
kubectl get svc
kubectl get hpa

text

Access the Flask service:

minikube service flask-service --url

Example output: http://127.0.0.1:54771
text

Test:

curl http://127.0.0.1:54771/
curl -X POST -H "Content-Type: application/json" -d "{"sampleKey":"from-k8s"}" http://127.0.0.1:54771/data
curl http://127.0.0.1:54771/data

text

---

## 7. DNS resolution explanation

Inside Kubernetes, CoreDNS creates DNS records for Services. The MongoDB Service is named `mongo` in the `default` namespace, so its DNS name is `mongo` or `mongo.default.svc.cluster.local`. The Flask pods use `MONGO_HOST=mongo`, and when they connect to `mongo:27017`, the name is resolved by Kubernetes DNS to the Mongo Service IP, which forwards traffic to the StatefulSet pod `mongo-0`. This enables stable inter‑pod communication without hard‑coding pod IPs. [file:223][web:12][web:16]

---

## 8. Resource requests and limits explanation

Each container specifies:

resources:
requests:
cpu: "0.2"
memory: "250Mi"
limits:
cpu: "0.5"
memory: "500Mi"

text

- **Requests** tell the scheduler the minimum CPU and memory required so it can place the pod on a node with enough capacity.  
- **Limits** cap the maximum CPU and memory the pod may use; if it exceeds the memory limit it may be OOMKilled, and if it exceeds CPU limit it will be throttled. This prevents a single workload from starving the cluster. [file:223][web:16]

---

## 9. Design choices

- **MongoDB StatefulSet + headless Service**: provides stable identities and persistent storage, better for stateful databases than a Deployment.  
- **Mongo Service internal only**: Mongo is only needed by Flask, so it is not exposed externally, reducing attack surface.  
- **Flask Deployment with 2+ replicas**: meets the requirement and provides basic high availability and fault tolerance.  
- **NodePort for Flask on Minikube**: simple way to access the app from the host; in production this would typically be a LoadBalancer or Ingress.  
- **HPA on CPU (70% target, 2–5 replicas)**: uses built‑in CPU metrics for straightforward autoscaling.  
- **Secrets for DB auth**: credentials stored in a Secret and injected via env vars, avoiding hard‑coding.  
- **Resource requests/limits**: chosen exactly as specified to guarantee predictable resource usage and stability.

---

## 10. Testing scenarios (autoscaling + DB)

### 10.1 Database interaction

1. Get service URL:

URL=$(minikube service flask-service --url)

text

2. Test:

curl "$URL/"
curl -X POST -H "Content-Type: application/json" -d '{"sampleKey":"from-k8s"}' "$URL/data"
curl "$URL/data"

text

This confirms that data is written to and read from MongoDB through the Flask API.

### 10.2 Autoscaling (cookie point)

1. Generate load (PowerShell example):

while ($true) {
curl.exe http://127.0.0.1:54771/ > $null
}

text

(Replace with the actual URL from `minikube service`.)

2. Watch HPA:

kubectl get hpa flask-app-hpa -w

text

3. Watch pods:

kubectl get pods -w

text

Under sustained load, CPU utilization rises above the 70% target and the HPA scales the `flask-app` Deployment from 2 to more replicas; once the load stops, it scales back down to 2 replicas. Screenshots:

- `autoscaling-load.png` – load loop running.  
- `autoscaling-hpa.png` – `kubectl get hpa -w` while scaling.  
- `autoscaling-pods.png` – `kubectl get pods -w` as new pods are created.