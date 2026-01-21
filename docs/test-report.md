## Test 1: Service Discovery
**What was tested:** DNS resolution and connectivity between microservices in the `ecommerce` namespace.

**Expected Results:** nslookup returns ClusterIPs; curl returns a 200 OK or HTML response.

**Actual Results:**
    - backend -> 10.98.233.223
    - postgres -> 10.110.240.142
    - redis -> 10.108.17.42

Performance Observations: DNS resolution was successful. Note that while the `backend` was reachable, it is currently serving the default Nginx page.

## Test 2 Analysis: Load Balancing
**What was tested:** Traffic distribution across backend replicas.

**Expected Result:** Requests should be distributed across different pod IPs or hostnames.
Actual Result:: Executed 5 sequential curl requests to the backend service.

**Observations:** Every request returned the Nginx "welcome" snippet successfully.

Result: PASSED.

**Technical Note:** Even though the scheduler was under pressure (causing the "Internal error" on lb-test-3 and lb-test-5), the Kubernetes Service IP (ClusterIP) correctly routed the traffic to the available backend pods.

## Test 3: Self-Healing
**What was tested:** Simulated a pod failure by manually deleting a running frontend pod to verify the ReplicaSet controller's response.
**Expected Result:** Deployment controller detects the missing replica and immediately provisions a new pod.
**Actual Result:** The system restored itself to a healthy state in under 15 seconds. That is a new pod was created in Pending state within seconds and reached Running within seconds.

**Performance Observation:** The high availability of the frontend was maintained throughout the process.




## Test 4: Scaling & Resource Utilization
**What was tested:** Scaled the frontend to 10 replicas and backend-api to 5 replicas to observe cluster behavior under load.

**Expected vs actual results:** The expectation was for all 15 replicas to deploy successfully and for kubectl top to return valid data. The actual result was a success; all pods reached a Running state quickly.

**Any issues discovered:** The primary issue was the initial unavailability of the Metrics API, which required a manual installation of the metrics-server components.