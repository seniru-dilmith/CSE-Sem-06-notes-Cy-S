> [!abstract] Executive Summary
> This note provides a comprehensive deep dive into the <span style="color: #2ecc71;">Kubernetes (K8s)</span> ecosystem, tracking its evolution from physical hardware to modern container orchestration. It covers core architecture, object management, advanced scheduling, networking, security, and the deployment of a real-world microservices application called <b style="color: #e67e22;">Game Hub</b>.

---

## <span style="color: #3498db;">1. Computing Evolution: Why Kubernetes?</span>

<b style="color: #3498db;">The Monolithic Era:</b> Decades ago, enterprises ordered hardware (IBM, etc.) and built on-premise data centers. Network engineers set up racks, and sysadmins installed OS and libraries (e.g., JDK for Java apps). 
- <b style="color: #e74c3c;">Limitations:</b> One change to a login endpoint required rebuilding the entire monolith. If one feature failed, everything had to be reverted.

<b style="color: #3498db;">The Virtualization Era (Circa 1960s/VMware):</b> Slicing hardware into <b style="color: #2ecc71;">Virtual Machines (VMs)</b> using Hypervisors (Type 1 or Type 2).
- <b style="color: #2ecc71;">Success:</b> Improved resource utilization from 20% to full capacity.

<b style="color: #3498db;">The Cloud Era:</b> Providers like <b style="color: #3498db;">AWS, GCP, Azure, and Exoscale</b> abstracted away data center complexity. Racks are now ordered via UI/CLI and delivered in minutes.

<b style="color: #3498db;">The Container Era (2013 - Docker):</b> Containers are slices of the Operating System, not the hardware. They package code and dependencies to run as isolated Linux processes.

> [!help] FAQ: What is Kubernetes, and what is the history of the computing context that led to its creation?
> <b style="color: #3498db;">Answer:</b> Kubernetes (often abbreviated as "K8s") is an open - source platform that automates the deployment, scaling, and management of containerized applications. Historically, organizations deployed monolithic applications directly on bare - metal physical servers, which was slow and resulted in wasted resources. The industry then evolved to virtual machines (virtualization), and subsequently to cloud computing, allowing users to rent infrastructure dynamically. Ultimately, the shift to microservices led to the container era (spearheaded by Docker, which uses Linux namespaces and cgroups for isolation), making it necessary to have a robust orchestrator like Kubernetes to manage hundreds of independent, containerized services.



---

## <span style="color: #3498db;">2. Kubernetes Architecture: The Engine Room</span>

Kubernetes operates on a <b style="color: #3498db;">Control Plane</b> and <b style="color: #3498db;">Worker Node</b> model.

### <span style="color: #2ecc71;">A. The Control Plane (The Brain)</span>
The control plane manages the cluster state and consists of:
1. <b style="color: #3498db;">API Server:</b> The central gateway. Every request (kubectl/curl) goes here first. It performs the "3 As" - Authentication, Authorization, and Admission.
2. <b style="color: #3498db;">etcd:</b> A highly available key - value store. It is the cluster's database. If it isn't in etcd, it doesn't exist in the cluster.
3. <b style="color: #3498db;">Scheduler:</b> Watches for new pods and finds the "feasible" node based on filtering and scoring.
4. <b style="color: #3498db;">Controller Manager:</b> The "watchdog" that ensures the <span style="color: #2ecc71;">Live State</span> matches the <span style="color: #3498db;">Desired State</span>.

> [!help] Question: What are the main components of the Kubernetes Control Plane?
> <b style="color: #3498db;">Answer:</b> The control plane manages the cluster and consists of:
> - <b style="color: #3498db;">API Server:</b> The central gateway that authenticates, authorizes, and admits all requests.
> - <b style="color: #3498db;">etcd:</b> A highly available key - value store that acts as the cluster's backend database, saving the state of the system.
> - <b style="color: #3498db;">Scheduler:</b> Finds the most feasible worker node for new pods based on filtering, scoring, and resource availability.
> - <b style="color: #3498db;">Controller Manager:</b> Continuously watches the cluster to ensure the actual state matches the desired state.

### <span style="color: #2ecc71;">B. The Worker Node (The Muscle)</span>
Worker nodes run the actual workloads.
1. <b style="color: #3498db;">Kubelet:</b> The agent running on each node. It receives pod specs from the API server and ensures containers are running.
2. <b style="color: #3498db;">Kube-proxy:</b> Manages network rules (IP tables/IPVS) to route traffic to pods.
3. <b style="color: #3498db;">Container Runtime:</b> Usually <b style="color: #3498db;">containerd</b>.

> [!help] FAQ: What are the main components of a Worker Node, including the extensibility interfaces?
> <b style="color: #3498db;">Answer:</b> The worker node runs the actual workloads. Its core component is the kubelet, which receives instructions to run pods. It interacts with three key extensibility interfaces:
> - <b style="color: #3498db;">CRI (Container Runtime Interface):</b> Pulls images and runs the containers (e.g., containerd) using Linux namespaces and cgroups.
> - <b style="color: #3498db;">CNI (Container Network Interface):</b> Connects the pod to the cluster network and assigns IP addresses via a bridge.
> - <b style="color: #3498db;">CSI (Container Storage Interface):</b> Manages attaching, mounting, and provisioning persistent storage volumes.
> Additionally, the node runs kube - proxy, which manages IP table rules to route traffic to the correct pods.

---

## <span style="color: #3498db;">3. The Extensibility Interfaces: CRI, CNI, CSI</span>

```mermaid
graph TD
    K[Kubelet] --> CRI[CRI: Container Runtime]
    K --> CNI[CNI: Networking]
    K --> CSI[CSI: Storage]
    CRI --> Container[Linux Process/Namespaces]
    CNI --> Veth[Veth Pairs/Bridge]
    CSI --> PV[Persistent Volumes]
````

### I. CRI (Container Runtime Interface)

CRI manages the pod container lifecycle. When Kubelet wants to run a pod:

- It calls containerd (CRI Implementation).
    
- containerd calls containerd-shim.
    
- containerd-shim calls runC (low - level runtime).
    
- runC creates the Linux namespaces (mnt, pid, net, uts, ipc, user) and cgroups.
    

### II. CNI (Container Network Interface)

Kubernetes does not have built - in networking. You must attach a plugin like Flannel, Calico, or Cilium.

- CNI assigns the pod IP.
    
- It creates the veth pair (one end in pod net namespace, one end on host).
    
- It connects the host side to a bridge (CNI0).
    

> [!note] The Pause Container
> 
> Every pod has a pause container. It does nothing but "sleep" and hold the network namespace open so other containers in the pod can share it and talk via localhost.

### III. CSI (Container Storage Interface)

Standardizes how storage (AWS EBS, NFS, Exoscale) connects to K8s.

- Provisioner: Watches PVCs and creates the cloud volume.
    
- Attacher: Attaches the volume to the node.
    
- Node Plugin: Mounts the volume into the pod path.
    

---

## 4. Deploying the Game Hub Application

Game Hub is a multi - tier microservices app consisting of:

- Front-end: React/Next.js (Port 80).
    
- Auth Service: Python (Port 8080).
    
- Game Service: Python (Port 8081).
    
- Database: PostgreSQL (Managed by Cloud Native PG).
    

### The Auth Service Deployment Example

YAML

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: crash-course
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
      - name: auth-service
        image: your-repo/auth-service:final
        ports:
        - containerPort: 8080
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-app
              key: username
```

---

## 5. Kubernetes Objects & Workloads

> [!help] Question: What are the different workload types in Kubernetes?
> 
> Answer:
> 
> - Pods: The smallest deployable unit, consisting of one or more containers sharing a network and file system.
>     
> - Deployments: Used for stateless applications; they automatically manage replica counts and facilitate rolling updates or rollbacks without downtime.
>     
> - ReplicaSets: A building block for Deployments that ensures a specified number of pod replicas are running.
>     
> - StatefulSets: Used for stateful applications (like databases). They provide sticky, predictable pod identities (e.g., postgres - 0, postgres - 1) and ordered scaling, making data replication and master/slave architectures easier.
>     
> - DaemonSets: Ensures that a copy of a specific pod runs on every node (or a subset of nodes), which is ideal for logging or monitoring agents.
>     

### A. The Pod Lifecycle

1. Pending: Waiting for scheduling or volume attachment.
    
2. Container Creating: Pulling images.
    
3. Running: Pod is active.
    
4. CrashLoopBackOff: Application keeps crashing.
    
5. Succeeded/Completed: Container ran to completion (e.g., a Job).
    

### B. Probes (The Health Checks)

- Startup Probe: Disables other probes until the app is fully started.
    
- Liveness Probe: If this fails, Kubelet kills and restarts the container.
    
- Readiness Probe: If this fails, the pod is removed from the Service endpoints (no traffic).
    

> [!important] Update Strategy
> 
> By default, Deployments use RollingUpdate.
> 
> - MaxSurge: How many extra pods can be created during update.
>     
> - MaxUnavailable: How many pods can be down during update.
>     

---

## 6. Advanced Scheduling

The Scheduler uses Filters (can the pod fit?) and Scoring (which node is the best fit?).

### Methods to Influence Scheduling:

1. nodeName: Bypasses scheduler, forces pod to a specific node.
    
2. nodeSelector: Simple label match (e.g., `workload: wasm`).
    
3. Node Affinity: Expressive rules.
    
    - RequiredDuringSchedulingIgnoredDuringExecution: Hard requirement.
        
    - PreferredDuringSchedulingIgnoredDuringExecution: Soft "try your best" requirement.
        
4. Taints & Tolerations: Nodes use Taints to repel pods. Pods use Tolerations to be allowed on tainted nodes.
    

> [!help] FAQ: What is the difference between Affinity and Taints/Tolerations?
> 
> Answer: Affinity is a property of the pod that attracts it to certain nodes. Taints are applied to the node to repel pods. A pod will only be scheduled on a tainted node if the pod explicitly has a matching Toleration.

> [!help] FAQ: What are Topology Spread Constraints?
> 
> Answer: They control how pods are distributed evenly across your cluster (by zones, racks, or nodes) to ensure high availability. You set a maxSkew to limit how lopsided the distribution of pods can be between different topologies.

---

## 7. Networking & Service Discovery

Pods are ephemeral; their IPs change. Services provide a persistent virtual IP (VIP).

### Service Types:

- ClusterIP (Default): Internal cluster communication.
    
- NodePort: Exposes service on a static port (30000 - 32767) on each Node's IP.
    
- LoadBalancer: Provisions a cloud load balancer (e.g., on AWS/Exoscale).
    
- Headless (ClusterIP: None): Returns pod IPs directly via DNS. Used for stateful apps.
    

> [!help] Question: How does CoreDNS handle service discovery?
> 
> Answer: CoreDNS is a DNS server that automatically resolves Kubernetes Service names to their internal virtual IP addresses. It allows pods to communicate with one another using fully qualified domain names (like `myservice.default.svc.cluster.local`) instead of relying on ephemeral pod IP addresses.

### The Gateway API: The Ingress Killer

Ingress is limited by its reliance on provider - specific annotations. The Gateway API is the modern evolution.

- GatewayClass: Controlled by infrastructure admins.
    
- Gateway: Controlled by cluster operators.
    
- HTTPRoute: Controlled by application developers.
    

> [!success] Check
> 
> We used K Gateway and Cert-Manager to deploy Game Hub with HTTPS on `k8225.cubsimplify.com`.

---

## 8. Configuration & Secrets Management

> [!help] Question: How do ConfigMaps and Secrets differ?
> 
> Answer: ConfigMaps store non - confidential configuration data (like environment variables or configuration files). Secrets store sensitive data (like passwords, tokens, or TLS certificates). By default, Secrets are only base64 - encoded, not encrypted, so they require additional configuration (like KMS providers) to encrypt at rest.

### Encryption at Rest

In production, you must enable encryption for etcd.

1. Create an `EncryptionConfiguration` file.
    
2. Update the API Server manifest (`/etc/kubernetes/manifests/kube-apiserver.yaml`).
    
3. Add the encryption provider flag.
    

---

## 9. Security & The "3 As"

> [!help] FAQ: What are the "3 A's" of Kubernetes API access?
> 
> Answer:
> 
> - Authentication: Verifying who the user or service account is (via tokens, OIDC, or client certificates). Anonymous requests should be explicitly disabled.
>     
> - Authorization: Checking if the entity is allowed to perform the action. This is primarily handled via Role - Based Access Control (RBAC) using Roles and RoleBindings (namespace - scoped) or ClusterRoles and ClusterRoleBindings (cluster - scoped).
>     
> - Admission: Webhooks that intercept requests to validate or mutate them (e.g., enforcing security policies) before they are saved to the database.
>     

> [!danger] Pod Security Warning
> 
> Pods should run as non-root users and use read-only file systems where possible to prevent container breakout attacks.

---

## 10. Observability & Troubleshooting

### Common Errors:

- ImagePullBackOff: Wrong image name, tag, or missing private registry credentials.
    
- CreateContainerConfigError: Missing ConfigMap or Secret.
    
- OOMKilled: Container exceeded its memory limit.
    
- Pending: No node has enough resources (CPU/RAM).
    

### Monitoring Stack:

- Prometheus: Scrapes metrics from `/metrics` endpoints.
    
- Grafana: Visualizes metrics in dashboards.
    
- ServiceMonitor: A Custom Resource that tells Prometheus which services to scrape.
    

---

## 11. GitOps: The Modern Way to Deploy

> [!help] Question: What is GitOps, and what are its core concepts?
> 
> Answer: GitOps is an operational framework where your version control system (like Git) serves as the single source of truth for your cluster's desired state. Core concepts include:
> 
> - Desired state: What the cluster should look like, defined by manifests in Git.
>     
> - Live state: The actual current state of the cluster.
>     
> - Reconciliation & Sync: The continuous process where a GitOps agent compares the desired state to the live state and applies changes to match them.
>     
> - Drift: When the live state deviates from the desired state.
>     
> - Pull - based delivery: The agent pulls changes from Git into the cluster, rather than traditional CI/CD which pushes changes.
>     

---

## 12. The Future: AI on Kubernetes

> [!help] Question: Why are AI platforms migrating to Kubernetes?
> 
> Answer: AI workloads have evolved from simple stateless inference to stateful, autonomous "agents" (using frameworks like LangGraph) that require long - running reasoning loops, durable execution, and state management. Kubernetes natively provides the necessary event - driven autoscaling (via tools like KEDA), persistent volumes, and security boundaries, turning it into the foundational operating system for end - to - end AI systems.

---

> [!done] Final Conclusion
> 
> Kubernetes is the operating system of the cloud. By mastering these constructs - architecture, networking, storage, and security - you can build production - grade environments that are resilient, scalable, and secure. Next steps: Explore Thanos for HA Monitoring and KEDA for advanced autoscaling.