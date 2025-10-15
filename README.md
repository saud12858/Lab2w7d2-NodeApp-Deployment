# Deploy a Standalone Node.js App on AKS (or Minikube)

This guide documents the full process of packaging, deploying, and exposing a simple Node.js web app using Kubernetes.

---

## 🧩 1. Clone Repository & Run Locally
```bash
sudo apt update -y
sudo apt install -y git nodejs npm
git clone https://github.com/saurabhd2106/sample-node-app-ci-lab-ih.git
cd sample-node-app-ci-lab-ih
npm install
npm start
curl -I http://localhost:3000/health  # should return 200 OK
```

---

## 🐳 2. Containerize App (Docker)
Create `Dockerfile`:
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app .
EXPOSE 3000
CMD ["node","server.js"]
```

Build and test:
```bash
docker build -t saud12858/node-web:v1 .
docker run -d -p 3000:3000 --name node-test saud12858/node-web:v1
curl -I http://localhost:3000/health
docker stop node-test && docker rm node-test
```

---

## ☁️ 3. Push Image to Docker Hub
```bash
docker login
docker push saud12858/node-web:v1
```

---

## 🧱 4. Namespace & ConfigMap
```bash
kubectl create ns node-challenge
```
`configmap.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: node-config
  namespace: node-challenge
data:
  BANNER: "Welcome to Saud's Node App 🚀"
```
Apply:
```bash
kubectl apply -f configmap.yaml
```

---

## 🚀 5. Deployment (3 Replicas)
`deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-web
  namespace: node-challenge
  labels:
    app: node-web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: node-web
  template:
    metadata:
      labels:
        app: node-web
    spec:
      containers:
        - name: node-web
          image: saud12858/node-web:v1
          ports:
            - containerPort: 3000
          env:
            - name: BANNER
              valueFrom:
                configMapKeyRef:
                  name: node-config
                  key: BANNER
          livenessProbe:
            httpGet: { path: /health, port: 3000 }
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet: { path: /health, port: 3000 }
            initialDelaySeconds: 3
            periodSeconds: 5
          resources:
            requests: { cpu: "50m", memory: "128Mi" }
            limits: { cpu: "250m", memory: "256Mi" }
```
```bash
kubectl apply -f deployment.yaml
kubectl get pods -n node-challenge -o wide
```

---

## 🌐 6. Services (ClusterIP + LoadBalancer)

**ClusterIP**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: node-internal
  namespace: node-challenge
spec:
  type: ClusterIP
  selector:
    app: node-web
  ports:
    - port: 3000
      targetPort: 3000
```

**LoadBalancer**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: node-public
  namespace: node-challenge
spec:
  type: LoadBalancer
  selector:
    app: node-web
  ports:
    - port: 80
      targetPort: 3000
```
Apply and check:
```bash
kubectl apply -f svc-clusterip.yaml -f svc-lb.yaml
kubectl get svc -n node-challenge -o wide
```
In Minikube:
```bash
minikube service node-public -n node-challenge --url
curl -I http://<url>/health
```

---

## ⚙️ 7. Scaling & Rollout
```bash
kubectl scale deploy/node-web -n node-challenge --replicas=5
kubectl get pods -n node-challenge -o wide
kubectl set image deploy/node-web node-web=saud12858/node-web:v2 -n node-challenge
kubectl rollout status deploy/node-web -n node-challenge
curl -I http://<url>/health
```

---

## 🧹 8. Cleanup
```bash
kubectl delete ns node-challenge
```

---

## ✅ Verification Checklist
- `/health` returns 200 OK internally and externally.  
- Pods = 3 (then 5 after scaling).  
- Rolling update completes with no downtime.  
- `kubectl get svc -o wide` shows ClusterIP and LoadBalancer.  
