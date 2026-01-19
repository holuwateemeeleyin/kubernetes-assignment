# kubernetes-assignment
Kubernetes Practical Assignment: E Commerce Platform Deployment Assignment Overview
## Assignment Overview
Duration: 4-6 hours
Group Size: 3 people
Difficulty: Intermediate
Prerequisites: Completed Kubernetes installation, understanding of pods, deployments, replica sets, taints, tolerations, node selectors, affinity, scheduling, and static pods.

## Scenario
Your team has been hired to deploy a microservices-based e-commerce platform on Kubernetes. The platform consists of multiple services with different requirements:
* Frontend - Customer-facing web application
* Backend API - Business logic and data processing
* Database - PostgreSQL database
* Cache - Redis cache layer
* Monitoring - Custom monitoring agent (static pod)

Each service has specific placement, scaling, and resource requirements that you must implement using Kubernetes scheduling features.

## Learning Objectives
By completing this assignment, you will:
1. Deploy a multi-tier application on Kubernetes
2. Implement node affinity and anti-affinity rules
3. Use taints and tolerations for specialized workloads
4. Configure node selectors for environment separation
5. Create and manage static pods
6. Scale deployments and understand replica sets
7. Troubleshoot common Kubernetes issues
8. Use kubectl effectively for cluster management


# Part 1: Cluster Setup and Node Labeling (30
minutes)
## Task 1.1: Install Kubernetes Cluster
Using the kubernetes-installation-guide.md , set up a 3-node Kubernetes cluster on AWS:
1 Control Plane node
2 Worker nodes
### Deliverables:
```
# Verify all nodes are Ready
kubectl get nodes

# Expected output:
# NAME STATUS ROLES AGE VERSION
# control-plane Ready control-plane 10m v1.35.x
# worker-node-1 Ready <none> 5m v1.35.x
# worker-node-2 Ready <none> 5m v1.35.x
```

## Task 1.2: Label Your Nodes
Label your worker nodes to simulate different environments and hardware capabilities:
```
# Label worker-node-1 as production with SSD storage
kubectl label nodes <worker-node-1> environment=production
kubectl label nodes <worker-node-1> storage=ssd
kubectl label nodes <worker-node-1> tier=frontend

# Label worker-node-2 as production with HDD storage
kubectl label nodes <worker-node-2> environment=production
kubectl label nodes <worker-node-2> storage=hdd
kubectl label nodes <worker-node-2> tier=backend
```

Verification:
```
kubectl get nodes --show-labels
```

## Task 1.3: Apply Taints to Nodes
Apply taints to control workload placement:
```
# Taint worker-node-1 for frontend workloads only
kubectl taint nodes <worker-node-1> workload=frontend:NoSchedule
# Taint worker-node-2 for backend workloads only
kubectl taint nodes <worker-node-2> workload=backend:NoSchedule
```

Verification:
```
kubectl describe node <worker-node-1> | grep -i taint
kubectl describe node <worker-node-2> | grep -i taint
```
### Submission:
Screenshot of `kubectl get nodes` --show-labels
Screenshot of taint verification commands




# Part 2: Deploy the Database Layer (45 minutes)

## Task 2.1: Create a Namespace
Create a dedicated namespace for the application:
`kubectl create namespace ecommerce`
Set this as your default namespace:
`kubectl config set-context --current --namespace=ecommerce`

##Task 2.2: Deploy PostgreSQL Database
Create a file named `postgres-deployment.yaml` :
Requirements:
Use `postgres:16` image
Set environment variable `POSTGRES_PASSWORD=secure_password`
Use node selector to run on the backend tier
Add toleration for backend taint
Set resource requests: CPU 500m, Memory 512Mi
Set resource limits: CPU 1000m, Memory 1Gi
Create a single replica
Expose port 5432

Hints:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: ecommerce
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      # Add node selector here
      # Add tolerations here
      containers:
        - name: postgres
          image: postgres:14
          # Complete the rest
```


## Task 2.3: Create a Service for PostgreSQL
Create `postgres-service.yaml` :
Requirements:
- Service type: ClusterIP
- Port: 5432
- Selector: app=postgres

## Task 2.4: Verify Database Deployment
```
# Apply the manifests
kubectl apply -f postgres-deployment.yaml
kubectl apply -f postgres-service.yaml

# Verify deployment
kubectl get pods -o wide
kubectl get svc
kubectl describe pod <postgres-pod-name>
```

### Questions to Answer:
1. On which node did the postgres pod get scheduled? Why?
2. What happens if you remove the toleration? Test it and explain.
3. Can you access the database from outside the cluster? Why or why not?

### Submission:
* `postgres-deployment.yaml` file
* `postgres-service.yaml` file
* Screenshot showing pod running on the correct node
* Answers to the questions

# Part 3: Deploy Redis Cache with Affinity Rules(45 minutes)

## Task 3.1: Deploy Redis
Create `redis-deployment.yaml` :

### Requirements:
- Use `redis:7-alpine` image
- Create 2 replicas for high availability
- Use pod anti-affinity to ensure replicas run on different nodes
- Add toleration to run on both frontend and backend nodes
- Set resource requests: CPU 250m, Memory 256Mi
- Expose port 6379

### Pod Anti-Affinity Configuration:
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app
              operator: In
              values:
                - redis
        topologyKey: "kubernetes.io/hostname"
```

## Task 3.2: Create Redis Service
Create `redis-service.yaml` :

Requirements:
- Service type: ClusterIP
- Port: 6379
- Selector: app=redis

## Task 3.3: Verify Redis Deployment
```
kubectl apply -f redis-deployment.yaml
kubectl apply -f redis-service.yaml

# Check which nodes the pods are running on
kubectl get pods -o wide -l app=redis
```

### Questions to Answer:
1. Are the Redis pods running on different nodes? Why?
2. What would happen if you tried to scale to 3 replicas but only have 2
worker nodes?
3. Change the anti-affinity rule to `preferredDuringSchedulingIgnoredDuringExecution`. What's the difference?

### Submission
`redis-deployment.yaml` file
`redis-service.yaml` file
Screenshot showing pods on different nodes
Answers to the questions

# Part 4: Deploy Backend API with Node Affinity (60 minutes)

## Task 4.1: Deploy Backend API
Create `backend-deployment.yaml` :

### Requirements:
- Use `nginx:alpine` image (as a placeholder for your backend)
- Create 3 replicas
- Use node affinity to prefer nodes with SSD storage but require backend tier
- Add toleration for backend workload
- Set resource requests: CPU 200m, Memory 256Mi
- Expose port 80
- Add environment variables:
  - DATABASE_HOST=postgres
  - REDIS_HOST=redis

Node Affinity Example:

affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: tier
              operator: In
              values:
                - backend
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
            - key: storage
              operator: In
              values:
                - ssd

## Task 4.2: Create Backend Service
Create `backend-service.yaml` :

### Requirements:
- Service type: ClusterIP
- Port: 8080 (target port 80)
- Selector: app=backend

## Task 4.3: Test Backend Connectivity
```
# Create a test pod to verify connectivity
kubectl run test-pod --image=curlimages/curl --rm -it --restart=Never -- sh

# Inside the pod, test connections:
curl http://backend:8080
nslookup postgres
nslookup redis
```
### Questions to Answer:
1. On which node(s) are the backend pods scheduled? Why?
2. What's the difference between requiredDuringScheduling and preferredDuringScheduling ?
3. Can the backend pods communicate with postgres and redis? Demonstrate.

### Submission:
- `backend-deployment.yaml` file
- `backend-service.yaml` file
- Screenshot of connectivity tests
- Answers to the questions

# Part 5: Deploy Frontend with Multiple Replicas (45 minutes)
## Task 5.1: Deploy Frontend Application
Create `frontend-deployment.yaml` :
### Requirements:
Use `nginx:alpine` image
Create 4 replicas
Use node selector for frontend tier
Add toleration for frontend workload
Set resource requests: CPU 100m, Memory 128Mi
Expose port 80
Add environment variable: `BACKEND_URL=http://backend:8080`

## Task 5.2: Create Frontend Service
Create `frontend-service.yaml` :
### Requirements:
Service type: NodePort
Port: 80
NodePort: 30080
Selector: app=frontend

## Task 5.3: Test Frontend Access
```
# Get the external IP of any worker node
kubectl get nodes -o wide

# Access the frontend (replace with your node IP)
curl http://<NODE_IP>:30080
```

## Task 5.4: Scale the Frontend
```
# Scale frontend to 6 replicas
kubectl scale deployment frontend --replicas=6

# Watch the scaling process
kubectl get pods -l app=frontend -w
```
### Questions to Answer:
1. Where are all the frontend pods scheduled? Why?
2. What happens if you try to scale to 10 replicas on a single node?
3. How does the NodePort service distribute traffic across replicas?
4. Access the application from your browser using http://<NODE_IP>:30080. Does it work?

### Submission:
- `frontend-deployment.yaml` file
- `frontend-service.yaml` file
- Screenshot showing 6 replicas running
- Screenshot of accessing the application
- Answers to the questions
