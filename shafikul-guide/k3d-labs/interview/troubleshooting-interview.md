# Kubernetes Troubleshooting Mental Model & Interview Prep

When diagnosing issues in Kubernetes, rely on the **Desired State vs. Actual State** and the **Chain of Command**.

## The 4-Step Framework

1. **The "What" (`kubectl get`)**: Observe the current state.
2. **The "Why" (`kubectl describe` and `kubectl get events`)**: Look for events and reasons from K8s controllers.
3. **The "Inside" (`kubectl logs`)**: Investigate the application code.
4. **The "Chain of Command"**: Understand who is managing the resource (`HPA -> Deployment -> ReplicaSet -> Pod`).

---

## 15 Interview-Style Scenarios

### Q1: The App is Crashing Immediately
**Interviewer:** "You deploy a new version of an app, and `kubectl get pods` shows it's stuck in `CrashLoopBackOff`. How do you troubleshoot this?"
**Your Answer:** "First, I'd run `kubectl describe pod <pod-name>` to ensure the infrastructure successfully pulled the image and started the container. Since it's in CrashLoop, it likely started but the app code failed. My next immediate step is to check the application logs using `kubectl logs <pod-name> -p` (using the `-p` or `--previous` flag) to see the stack trace or error message that caused the container to exit before it restarted."

### Q2: The Pod Never Starts (Bad Image)
**Interviewer:** "A pod is stuck in `ImagePullBackOff`. What is happening and how do you fix it?"
**Your Answer:** "This means the Kubelet is failing to pull the container image from the registry. I would run `kubectl describe pod <pod-name>` and look at the Events section. It will tell me the exact reason: either the image tag is misspelled, it doesn't exist, or Kubernetes lacks the authentication (an `imagePullSecret`) needed to pull from a private registry."

### Q3: Missing Pods (The "Phantom" Scale Down)
**Interviewer:** "A developer complains that they set `replicas: 4` in their Deployment, but there are only 2 pods running. What could be causing this?"
**Your Answer:** "This is usually caused by a HorizontalPodAutoscaler (HPA) managing the deployment. The HPA overrides the Deployment's replica count based on metrics. I would run `kubectl get hpa` to see if one exists, and then `kubectl describe hpa <name>` to check its scaling events and see if it scaled down due to low CPU or memory usage."

### Q4: Pods are Running but App is Unreachable Internally
**Interviewer:** "Your pods are `Running` and `1/1` READY. However, when you curl the Service from another pod, it times out. How do you debug this?"
**Your Answer:** "I would first verify that the Service is successfully routing to the pods by running `kubectl get endpoints <service-name>`. If the endpoints list is empty, it means the Service's `selector` labels don't match the Pods' labels. If the endpoints *are* populated, I would check if a `NetworkPolicy` is blocking the traffic, or run `kubectl logs` to ensure the app inside the pod is actually listening on the correct port."

### Q5: Out of Memory (OOMKilled)
**Interviewer:** "A pod's status is `OOMKilled`. Why did this happen?"
**Your Answer:** "The container's process tried to consume more memory than was allocated in its `resources.limits.memory` specification. The Linux kernel (OOM Killer) terminated the process to protect the node. I would either need to fix a memory leak in the application code, or increase the memory limits in the pod spec."

### Q6: Pod Stuck in Pending (Insufficient Resources)
**Interviewer:** "You schedule a new pod, but it sits in the `Pending` state forever. What is the most likely cause?"
**Your Answer:** "The most common cause is that the Kubernetes Scheduler cannot find a node with enough available CPU or Memory to satisfy the pod's `resources.requests`. I would verify this by running `kubectl describe pod <pod-name>` and looking at the Events at the bottom for a `FailedScheduling` message."

### Q7: Readiness Probe Failing (Traffic Blackholing)
**Interviewer:** "A pod is `Running` but its status shows `0/1` READY. It is not receiving any web traffic. Why?"
**Your Answer:** "The pod is failing its Readiness Probe. Kubernetes starts the container, but the application is returning a non-200 status code or timing out on the readiness check endpoint. Because it is not 'ready', Kubernetes removes it from the Service's Endpoints so it won't receive user traffic. I would check `kubectl describe pod` to see the probe failure, and then check `kubectl logs` to see why the app isn't ready."

### Q8: Liveness Probe Failing (Constant Restarts)
**Interviewer:** "A pod keeps restarting every minute, but when you check the logs, the application seems to start up perfectly fine with no errors. What is happening?"
**Your Answer:** "It's highly likely the Liveness Probe is failing. The Kubelet pings the liveness endpoint, gets a failure (or timeout), assumes the app is deadlocked, and forcefully restarts the container. I would check `kubectl describe pod` for `Liveness probe failed` events, and consider increasing the `initialDelaySeconds` if the app just takes a long time to boot up."

### Q9: ConfigMap or Secret Missing
**Interviewer:** "A pod status is `CreateContainerConfigError`. What does this mean?"
**Your Answer:** "This means the pod is trying to mount a ConfigMap or Secret as a volume, or use it for environment variables, but that specific ConfigMap or Secret does not exist in the namespace. I would use `kubectl describe pod` to find out the exact name of the missing resource and create it."

### Q10: Persistent Volume Claim (PVC) Stuck in Pending
**Interviewer:** "A stateful pod is `Pending`, and you notice its PVC is also `Pending`. How do you investigate?"
**Your Answer:** "The pod can't be scheduled until its storage is ready. I would run `kubectl describe pvc <pvc-name>` and look at the events. Common issues include a missing or misspelled `StorageClass`, the cloud provider hitting a storage quota, or no PersistentVolumes being available to bind to."

### Q11: Pod Evicted by Node
**Interviewer:** "You see several pods with the status `Evicted`. What causes a pod to be evicted?"
**Your Answer:** "Eviction happens when the Node experiences resource pressure, typically running out of disk space (`DiskPressure`) or memory (`MemoryPressure`). The Kubelet starts terminating pods to save the node from crashing. I would run `kubectl describe pod` to confirm the eviction reason."

### Q12: Network Policy Blocking Traffic
**Interviewer:** "Two pods in different namespaces used to talk to each other, but suddenly connections are timing out. The Services and Endpoints are perfectly fine. What could be the issue?"
**Your Answer:** "If DNS and Endpoints are fine, it's likely a networking or firewall issue. Someone might have applied a `NetworkPolicy` with a default-deny rule that is now blocking ingress or egress traffic between those namespaces. I would check `kubectl get networkpolicies` in both namespaces."

### Q13: DNS Resolution Failing
**Interviewer:** "Your application logs show 'Unknown host: database-svc'. How do you verify if this is a Kubernetes DNS issue?"
**Your Answer:** "I would exec into a troubleshooting pod (like a busybox or alpine container) using `kubectl exec -it <pod> -- sh` and run `nslookup database-svc`. If it fails, I would then check if the `CoreDNS` pods in the `kube-system` namespace are running and healthy."

### Q14: Node Taints and Tolerations Mismatch
**Interviewer:** "You have a cluster with plenty of resources. You deploy a pod, but it stays `Pending`. `kubectl describe` shows a `Taints` related message. Explain."
**Your Answer:** "The nodes in the cluster have a 'Taint' applied to them (e.g., dedicated for GPU workloads). Since my pod specification does not have a matching 'Toleration', the scheduler is forbidding the pod from running on those nodes. I either need to add the toleration to my pod, or remove the taint from the node."

### Q15: Ingress Returning 502/504 Bad Gateway
**Interviewer:** "When users hit your application's public URL, they get a 502 Bad Gateway from the Ingress controller. How do you trace this down?"
**Your Answer:** "A 502 means the Ingress controller (like Nginx) received the request, but couldn't get a valid response from the backend Service. First, I'd check if the Ingress rule is pointing to the correct Service name and port. Second, I'd check `kubectl get endpoints` to ensure the Service actually has healthy pods attached to it. Finally, I'd check the application logs to see if it's dropping the connection."
