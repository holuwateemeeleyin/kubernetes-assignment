## Part 2: Deploy the Database Layer
### Questions to Answer:
1. On which node did the postgres pod get scheduled? Why?
Answer: 
The postgres pod was scheduled on worker-node-2.
Why?: This occurred because we configured a nodeSelector in the deployment YAML specifically looking for the label tier: backend. Since worker-node-2 is the only node with that label (and the designated HDD storage), the Kubernetes scheduler placed the pod there to meet that hardware requirement.

2. What happens if you remove the toleration? Test it and explain.
Answer: 
If the toleration is removed, the pod will get stuck in a Pending state and will never be scheduled.
Explanation: This happens because worker-node-2 has a Taint (workload=backend:NoSchedule). A taint acts like a "lock" that prevents pods from landing on a node unless they have a matching "key," which is the Toleration. Without that toleration in the YAML, the scheduler sees worker-node-2 as off-limits and, since no other nodes match the nodeSelector, the pod has nowhere to go.

3. Can you access the database from outside the cluster? Why or why not?
Answer: No, you cannot access it from outside the cluster.
Explanation: This is because we used a service of type: ClusterIP. This service type only provides an internal IP address that is reachable by other pods inside the Kubernetes cluster (like your backend API). It does not expose a port on the AWS EC2 instance's public IP address or an external Load Balancer, which keeps the database secure from outside traffic.



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
