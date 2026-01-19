## Part 2: Deploy the Database Layer

### Questions to Answer

1. **On which node did the postgres pod get scheduled? Why?**

   **Answer:**  
   The postgres pod was scheduled on `worker-node-2`.  

   **Why?:**  
   This occurred because we configured a `nodeSelector` in the deployment YAML specifically looking for the label `tier: backend`. Since `worker-node-2` is the only node with that label (and the designated HDD storage), the Kubernetes scheduler placed the pod there to meet that hardware requirement.

2. **What happens if you remove the toleration? Test it and explain.**

   **Answer:**  
   If the toleration is removed, the pod will get stuck in a **Pending** state and will never be scheduled.  

   **Explanation:**  
   Worker-node-2 has a **taint** (`workload=backend:NoSchedule`). A taint acts like a "lock" that prevents pods from landing on a node unless they have a matching **toleration**. Without that toleration in the YAML, the scheduler sees `worker-node-2` as off-limits, and since no other nodes match the `nodeSelector`, the pod has nowhere to go.

3. **Can you access the database from outside the cluster? Why or why not?**

   **Answer:**  
   No, you cannot access it from outside the cluster.  

   **Explanation:**  
   This is because we used a service of `type: ClusterIP`. This service type only provides an internal IP address that is reachable by other pods inside the Kubernetes cluster (like your backend API). It does not expose a port on the AWS EC2 instance's public IP address or an external Load Balancer, which keeps the database secure from outside traffic.



## Part 3: Deploy Redis Cache with Affinity Rules

### Questions to Answer

1. **Are the Redis pods running on different nodes? Why?**  

   **Answer:**  
   Yes, they are running on different nodes (one on `worker-node-1` and one on `worker-node-2`).  

   **Reason:**  
   This is due to the Pod Anti-Affinity rule configured with `requiredDuringSchedulingIgnoredDuringExecution`. This **hard rule** instructs the Kubernetes scheduler that it is mandatory to place pods with the label `app: redis` on different hostnames to ensure high availability.

2. **What would happen if you tried to scale to 3 replicas but only have 2 worker nodes?**  

   **Answer:**  
   Two pods would run successfully (one on each node), but the third pod would remain in a **Pending** state.  

   **Reason:**  
   Because the Anti-Affinity rule is "required," the scheduler will not place a second Redis pod on a node that already has one. Since there are no more unique nodes available, the third pod has nowhere to go without violating the rule.

3. **Change the anti-affinity rule to `preferredDuringSchedulingIgnoredDuringExecution`. What's the difference?**  

   **Answer:**  
   This changes the rule from a **hard requirement** to a **soft preference**.  

   **Impact:**  
   In this mode, the scheduler will try to place the pods on different nodes. However, if it cannot find an empty node (like in the 3-replica scenario above), it will place the pod on an existing node anyway rather than leaving it Pending.


## Part 4: Deploy Backend API with Node Affinity

### Questions to Answer

1. **On which node(s) are the backend pods scheduled? Why?**

    **Answer:**
    The backend pods are scheduled on worker-node-2.

    **Reason:**
    Although worker-node-1 has the preferred ssd storage, only worker-node-2 satisfies the mandatory `requiredDuringScheduling` rule for the tier: backend label.

2. **What's the difference between `requiredDuringScheduling` and `preferredDuringScheduling` ?**
    
    **Required:**
    This is a strict constraint (hard affinity). If no node matches the rule, the pod remains Pending.
    
    **Preferred:**
    This is a weighted preference (soft affinity). The scheduler will try to meet this rule to "score" nodes higher, but it will still schedule the pod on a non-matching node if necessary

3. Can the backend pods communicate with postgres and redis? Demonstrate

    **Answer:**
    Yes, the connectivity is successful.
    
    **Demonstration:**
    The nslookup results show that the internal Kubernetes DNS (CoreDNS) correctly maps the service names to their ClusterIPs, and the curl command confirms the backend service is reachable on port 8080.

## Part 5: Deploy Frontend with Multiple Replicas
### Questions to Answer

1. **Where are all the frontend pods scheduled? Why?**
    
    **Answer:**
    All pods are scheduled on worker-node-1.
    
    **Why:**
    Because of the nodeSelector in the deployment manifest which requires the tier: frontend label, which only exists on worker-node-1.

2. **What happens if you try to scale to 10 replicas on a single node?**
    
    **Answer:** 
    The pods might stay in a Pending state.
    
    **Why:**
    A single node has limited CPU and Memory; once the aggregate requests of the 10 pods exceed the node's capacity, the scheduler cannot place more pods there.

3. **How does the NodePort service distribute traffic across replicas?**

    **Answer:**
    It uses kube-proxy. Traffic hitting port 30080 on any node is intercepted and forwarded to the frontend pods, regardless of which node they are actually running on.

4. **Access the application from your browser using http://<NODE_IP>:30080. Does it work?**

    **Answer:**
    No, it(http://172.31.26.165:30080) does not work directly from our local computer's browser. This is because the IP is internal AWS IP. It's a private address that only exist inside AWS data center to lab network. To demonstrate it works, we use the curl command; That is `curl http://172.31.26.165:30080` from the Control Plane

