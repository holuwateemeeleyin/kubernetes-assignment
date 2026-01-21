# Ecommerce System Deployment Documentation

## 1. Cluster Architecture
The cluster consists of one Control-Plane and two Worker nodes. 
- **Worker-Node-1:** Dedicated to Frontend workloads using Node Affinity and Taints.
- **Worker-Node-2:** Dedicated to Backend and Database workloads.
- **Service Mesh:** Internal communication is handled via ClusterIP Services and CoreDNS.

## 2. Scheduling Decisions
- **Affinity/Anti-Affinity:** Used to ensure Frontend pods are distributed across nodes to prevent single points of failure.
- **Taints & Tolerations:** Applied to ensure only specific workloads land on designated nodes, preventing resource contention between the database and the web UI.
- **Priority Classes:** `high-priority` was assigned to the Frontend to ensure it remains available during resource pressure.

## 3. Challenges & Solutions
- **Challenge:** Test pods failing to schedule due to taints.
- **Solution:** Manually cleared "NotReady" taints and updated pod definitions to include necessary tolerations.
- **Challenge:** Metrics API unavailable for performance reporting.
- **Solution:** Deployed and patched the `metrics-server` with `--kubelet-insecure-tls`.

## 4. Best Practices Learned
- Always set resource requests/limits to avoid "OOMKilled" errors.
- Use namespaces to isolate environments.
- Monitoring is essential; without metrics-server, cluster health is invisible.

## 5. How to Deploy
1. Run `kubectl apply -f namespace.yaml`
2. Apply PriorityClasses and ResourceQuotas.
3. Deploy Databases (`postgres`, `redis`) first.
4. Deploy `backend-api` and `frontend` services and deployments.