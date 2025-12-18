# Flask + MongoDB on Kubernetes (Minikube)

This project implements the **FarAlpha Intern Assignment**: a Python Flask API backed by MongoDB, deployed on a local Kubernetes cluster (Minikube) with authentication, persistent storage, Services, Horizontal Pod Autoscaler (HPA), DNS‑based service discovery, and resource requests/limits.

---

## 1. Application Overview

### API Endpoints

* **GET /**
  Returns:

  ```text
  Welcome to the Flask app! The current time is: <Date and Time>
  ```

* **GET /data**
  Returns all documents from MongoDB (excluding `_id`).

* **POST /data**
  Inserts a JSON document into MongoDB.

The Flask app connects to MongoDB using **username/password from environment variables**, which are sourced from a **Kubernetes Secret**.

---

## 2. Local Development (Part 1)

> ⚠️ **Note:** Part 1 is not required for submission, but is included for completeness.

### 2.1 Setup

```bash
mkdir flask-mongodb-app
cd flask-mongodb-app

python -m venv venv
```

**Activate virtual environment**

```bash
# Windows
venv\Scripts\activate

# Linux / macOS
source venv/bin/activate
```

Install dependencies:

```bash
pip install -r requirements.txt
```

#### Virtual Environment Benefits (Cookie Point 🍪)

* Isolates dependencies per project.
* Prevents conflicts between different Python projects.
* Makes deployments reproducible and cleaner.

---

### 2.2 Run MongoDB Locally with Docker

```bash
docker pull mongo:latest
docker run -d -p 27017:27017 --name mongodb mongo:latest
```

Create a `.env` file:

```env
MONGODB_URI=mongodb://localhost:27017/
```

Run Flask (Windows example):

```bash
set FLASK_APP=app.py
set FLASK_ENV=development
flask run
```

Test locally:

```bash
curl http://localhost:5000/

curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"sampleKey":"sampleValue"}' \
  http://localhost:5000/data

curl http://localhost:5000/data
```

---

## 3. Flask Application (Kubernetes‑Ready)

### `app.py`

```python
from flask import Flask, request, jsonify
from pymongo import MongoClient
from datetime import datetime
import os

app = Flask(__name__)

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

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

---

## 4. Dockerfile and Image

### `Dockerfile`

```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

ENV FLASK_APP=app.py
ENV FLASK_ENV=production

EXPOSE 5000

CMD ["flask", "run", "--host=0.0.0.0", "--port=5000"]
```

### Build and Push Image

```bash
docker build -t tr3n/flask-mongodb-app:latest .
docker login
docker push tr3n/flask-mongodb-app:latest
```

---

## 5. Kubernetes Manifests

### 5.1 MongoDB Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secret
type: Opaque
stringData:
  mongo-root-username: root
  mongo-root-password: examplepassword
```

---

### 5.2 Storage (PV & PVC)

**PersistentVolume**

```yaml
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
```

**PersistentVolumeClaim**

```yaml
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
```

---

### 5.3 MongoDB Service & StatefulSet

**Headless Service**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo
spec:
  clusterIP: None
  selector:
    app: mongo
  ports:
    - port: 27017
      targetPort: 27017
```

**StatefulSet**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: mongo
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
```

---

### 5.4 Flask Deployment, Service & HPA

**Deployment**

```yaml
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
              value: mongo
            - name: MONGO_PORT
              value: "27017"
            - name: MONGO_DB
              value: flask_db
            - name: MONGO_AUTH_DB
              value: admin
          resources:
            requests:
              cpu: "0.2"
              memory: "250Mi"
            limits:
              cpu: "0.5"
              memory: "500Mi"
```

**Service**

```yaml
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
```

**Horizontal Pod Autoscaler**

```yaml
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
```

---

## 6. Deployment on Minikube

```bash
minikube start --memory=4096 --cpus=2
```

### Apply MongoDB Resources

```bash
kubectl apply -f mongo-secret.yaml
kubectl apply -f mongo-pv.yaml
kubectl apply -f mongo-pvc.yaml
kubectl apply -f mongo-service.yaml
kubectl apply -f mongo-statefulset.yaml
kubectl get pods
```

### Apply Flask Resources

```bash
kubectl apply -f flask-deployment.yaml
kubectl apply -f flask-service.yaml
kubectl apply -f flask-hpa.yaml

kubectl get pods
kubectl get svc
kubectl get hpa
```

Access the service:

```bash
minikube service flask-service --url
```

---

## 7. DNS Resolution Explanation

Kubernetes **CoreDNS** automatically creates DNS records for Services. Since MongoDB is exposed via a Service named `mongo`, Flask pods can connect using:

```text
mongo:27017
```

This resolves to:

```text
mongo.default.svc.cluster.local
```

Traffic is forwarded to the MongoDB StatefulSet pod (`mongo-0`), enabling stable communication without hard‑coding IP addresses.

---

## 8. Resource Requests & Limits

```yaml
resources:
  requests:
    cpu: "0.2"
    memory: "250Mi"
  limits:
    cpu: "0.5"
    memory: "500Mi"
```

* **Requests**: minimum resources guaranteed by the scheduler.
* **Limits**: maximum resources a container can consume.

This prevents a single workload from starving the cluster and ensures predictable behavior.

---

## 9. Design Choices

* MongoDB as a **StatefulSet + Headless Service** for stable identity and storage.
* MongoDB exposed **internally only** for security.
* Flask Deployment with **2+ replicas** for availability.
* **NodePort** for Minikube access (Ingress/LoadBalancer in production).
* **HPA** based on CPU utilization (70%).
* **Secrets** for database credentials.
* Explicit **resource requests and limits**.

---

## 10. Testing Scenarios

### 10.1 Database Interaction

```bash
URL=$(minikube service flask-service --url)

curl "$URL/"

curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"sampleKey":"from-k8s"}' \
  "$URL/data"

curl "$URL/data"
```

### 10.2 Autoscaling (Cookie Point 🍪)

Generate load (PowerShell example):

```powershell
while ($true) {
  curl.exe http://127.0.0.1:54771/ > $null
}
```

Watch scaling:

```bash
kubectl get hpa flask-app-hpa -w
kubectl get pods -w
```

During testing, the HPA remained at the minimum replica count (2) even under sustained load.
The HPA status showed `cpu: <unknown>/70%`, indicating that CPU metrics were not available.

This typically occurs when the Kubernetes Metrics Server is not enabled or is not providing metrics.
As a result, the HPA could not evaluate CPU utilisation and therefore did not scale the Deployment.

Despite this, the HPA configuration itself (min/max replicas, CPU target) was correct and
actively monitoring the Deployment.

