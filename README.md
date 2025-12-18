# Flask + MongoDB on Kubernetes (Minikube)

This project implements the FarAlpha intern assignment: a Python Flask API backed by MongoDB, deployed on a local Kubernetes cluster (Minikube) with authentication, persistent storage, Services, Horizontal Pod Autoscaler, DNS-based service discovery, and resource requests/limits.[1]

## Application overview

- `GET /` returns a welcome message with the current time.
- `GET /data` returns all documents from MongoDB (excluding `_id`).
- `POST /data` inserts a JSON document into MongoDB.

The Flask app reads MongoDB credentials from environment variables provided by a Kubernetes Secret.

## Docker image

```bash
docker build -t tr3n/flask-mongodb-app:latest .
docker login
docker push tr3n/flask-mongodb-app:latest
Kubernetes manifests
Main YAML files:

mongo-secret.yaml – Mongo root user/password (Secret)

mongo-pv.yaml, mongo-pvc.yaml – PersistentVolume and PVC for Mongo

mongo-service.yaml – internal/headless Service for Mongo

mongo-statefulset.yaml – MongoDB StatefulSet with authentication and volume

flask-deployment.yaml – Flask Deployment (2 replicas) with env vars and resources

flask-service.yaml – NodePort Service exposing Flask

flask-hpa.yaml – Horizontal Pod Autoscaler (min 2, max 5, target 70% CPU)

Deployment steps on Minikube
bash
# Start cluster
minikube start --memory=4096 --cpus=2

# MongoDB resources
kubectl apply -f mongo-secret.yaml
kubectl apply -f mongo-pv.yaml
kubectl apply -f mongo-pvc.yaml
kubectl apply -f mongo-service.yaml
kubectl apply -f mongo-statefulset.yaml

kubectl get pods

# Flask resources
kubectl apply -f flask-deployment.yaml
kubectl apply -f flask-service.yaml
kubectl apply -f flask-hpa.yaml

kubectl get pods
kubectl get svc
kubectl get hpa
Get the Flask service URL and test:

bash
minikube service flask-service --url
# Example: http://127.0.0.1:54771

curl http://127.0.0.1:54771/
curl -X POST -H "Content-Type: application/json" -d '{"sampleKey":"from-k8s"}' http://127.0.0.1:54771/data
curl http://127.0.0.1:54771/data
DNS resolution (summary)
Kubernetes CoreDNS creates DNS records for Services. The MongoDB Service is named mongo in the default namespace, so its DNS name is mongo (or mongo.default.svc.cluster.local). The Flask pods use MONGO_HOST=mongo; when they connect to mongo:27017, Kubernetes DNS resolves this name to the Mongo Service IP, which forwards traffic to the MongoDB pod.​

Resource requests and limits
Both Mongo and Flask containers use:

text
resources:
  requests:
    cpu: "0.2"
    memory: "250Mi"
  limits:
    cpu: "0.5"
    memory: "500Mi"
Requests are the minimum resources required for scheduling, while limits cap maximum usage to avoid any pod consuming too many cluster resources.​

Testing autoscaling
Load generator (PowerShell):

powershell
while ($true) {
  curl.exe http://127.0.0.1:54771/ > $null
}
With load running:

bash
kubectl get hpa flask-app-hpa -w
kubectl get pods -w
Under sustained load, HPA scales the flask-app Deployment from 2 replicas up, then back down when load stops. Screenshots:

autoscaling-load.png – curl loop running

autoscaling-hpa.png – kubectl get hpa -w

autoscaling-pods.png – kubectl get pods -w