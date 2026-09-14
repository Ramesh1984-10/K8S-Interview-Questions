# K8S-Interview-Questions

# Q1 Kubernetes Arch
  Ans: Master Node ( Kube Api, Etcd, Scheduler, Controller)
      - Worker Node (Kubelet, Kube-Proxy, Containerd)

# Q2 Replication Controller and ReplicaSet 
  Ans:  Replicaset set based and equality based both (matchExpression and matchLabels) while in Replication controller using only equality based (==, !=)

# Q3 Stateless vs State Full  Deployment in K8S
  Ans: 

  <img width="556" height="322" alt="image" src="https://github.com/user-attachments/assets/5b0cc315-2fb6-4e24-a4de-4a8a82a2e54c" />

# Q4 Storage Class 

  Ans: There are two types of Storage Manual and Dynamic Storage Class

  Instead of manually creating physical or cloud-based storage volumes ahead of time, a StorageClass acts as a blueprint for dynamic provisioning. When a user requests        storage, Kubernetes uses the StorageClass to automatically provision the underlying volume.

Key Concepts of a StorageClass
 - Provisioner: Determines what volume plugin is used to provision the PersistentVolume (PV). Examples include cloud-specific plugins (like AWS EBS, Google Cloud Persistent Disk) or Container Storage Interface (CSI) drivers.

 - Parameters: Describe the specific options for the storage backend. For instance, an AWS EBS storage class might specify whether the underlying disk type is gp3 or io1, along with IOPS configurations.

 - Reclaim Policy: Dictates what happens to the underlying storage when the associated PersistentVolumeClaim (PVC) is deleted.
      - Delete: Automatically deletes the storage asset in the underlying infrastructure.
      - Retain: Leaves the volume intact so data can be manually recovered.

 - Volume Binding Mode: Controls when dynamic provisioning and volume binding should occur.
    - Immediate: Happens as soon as the PVC is created.
    - WaitForFirstConsumer: Delays binding and provisioning until a Pod that uses the PVC is actually scheduled, ensuring the volume is created in the correct availability        zone/node topology.

    <img width="284" height="245" alt="image" src="https://github.com/user-attachments/assets/3ce26fa6-385e-4ed3-9155-581689bcf938" />



# Q5 There is an Pod and Node which Showing in Pending State , Give Reasons ?
  Ans: 
  **Top Reasons a Pod Remains in Pending State**

* **Resource Exhaustion:** There is not enough available CPU or memory across your nodes to satisfy the pod's requested resources. Verify this by checking the Events section of `kubectl describe pod <pod-name>` for a `FailedScheduling` message stating `Insufficient cpu` or `Insufficient memory`.
* **Unmet Taints and Tolerations:** The available nodes have taints applied (e.g., `NoSchedule`), and your pod lacks the corresponding tolerations to run on them. Verify the node taints by running `kubectl describe node <node-name> | grep Taints`.
* **Unbound Storage (PVC):** The pod is requesting a `PersistentVolumeClaim` that has not yet been dynamically provisioned or bound to a PersistentVolume. Verify that your storage is ready by running `kubectl get pvc` and ensuring the status is `Bound`.
* **Scheduling Constraints:** Strict `nodeSelector`, `nodeAffinity`, or `podAntiAffinity` rules are preventing the scheduler from finding a valid node match. Verify that the pod's node selector matches the exact labels on your target node using `kubectl get nodes --show-labels`.

**Top Reasons a Node Remains in Pending/NotReady State**

* **Missing CNI Plugin:** The Container Network Interface (such as Calico, Flannel, or AWS VPC CNI) is not installed or its pods are crashing, leaving the node without networking capabilities. Verify your network pods are running correctly by checking `kubectl get pods -n kube-system`.
* **Kubelet Failure:** The `kubelet` service on the worker node has crashed, failed to start, or has invalid configuration arguments. Verify this by connecting directly to the node via SSH and running `systemctl status kubelet`.
* **Cloud Provider IAM Issues:** In managed environments like AWS EKS, the underlying EC2 instance may lack the correct IAM instance profile to authenticate and register with the Kubernetes control plane. Verify that the IAM role attached to the EC2 instance contains the necessary EKS worker node policies (e.g., `AmazonEKSWorkerNodePolicy`).
* **Control Plane Disconnection:** Security groups, VPC routing rules, or firewalls are blocking the node from communicating with the Kubernetes API server. Verify that outbound traffic on port 443 (or 6443) is permitted from the worker node's subnet to the control plane.
* **System Resource Pressure:** The node is experiencing severe `DiskPressure`, `MemoryPressure`, or `PIDPressure`, forcing it out of a Ready state to protect core services. Verify node conditions by looking at the `Conditions` table in `kubectl describe node <node-name>`.


# Q6 What is advantage of calico network in Kubernetes and traffic Flow ?
  Ans: 
    **Advantages of Project Calico**

* **High Performance (No Encapsulation by Default):** Unlike CNI plugins that default to overlay networks (like Flannel with VXLAN), Calico primarily routes packets natively using standard Layer 3 routing via BGP (Border Gateway Protocol). This avoids the CPU overhead of wrapping and unwrapping packets, resulting in near bare-metal network throughput.
* **Advanced Network Policies:** While Kubernetes has standard NetworkPolicies, Calico provides extended Custom Resource Definitions (CRDs) such as `GlobalNetworkPolicy`. These allow administrators to enforce zero-trust security rules globally across all namespaces, filter traffic by service accounts, and apply rules to host endpoints (nodes) themselves.
* **eBPF Data Plane Option:** Modern Calico can replace standard Linux `iptables` with an eBPF (Extended Berkeley Packet Filter) data plane. This allows Calico to completely bypass `kube-proxy`, drastically reducing latency, automatically preserving client source IP addresses, and lowering CPU utilization for high-throughput environments.
* **Massive Scalability:** Because Calico relies on BGP—the same protocol used to route the internet—it can scale efficiently to clusters with tens of thousands of nodes without creating massive central routing bottlenecks.

**Calico Traffic Flow Mechanism**

* **Local Node Routing (veth pairs):** When a Pod is created, Calico assigns it an IP address and creates a virtual ethernet (`veth`) interface pair. One end sits inside the Pod, and the other connects to the host's root network namespace. Traffic is routed directly through the host's standard Linux routing table rather than a virtual bridge.
* **Cross-Node (Unencapsulated Layer 3):** When a Pod communicates with a Pod on a different node, the packet hits the host's routing table. The Calico `bird` daemon uses BGP to constantly share Pod IP routes with all other nodes in the cluster. The packet is sent directly to the destination node over the physical network as a standard IP packet.
* **Cross-Node (Encapsulated Overlay):** If the underlying cloud network (like AWS VPCs or Azure Vnets) blocks routing for unrecognized IP addresses, Calico can fall back to an overlay network using **IP-in-IP** or **VXLAN**. The original Pod-to-Pod packet is encapsulated inside a Node-to-Node packet to safely cross the network boundary, then decapsulated upon arrival.
* **Policy Enforcement Checkpoint:** Before a packet is allowed to leave the node, or immediately upon arriving at the destination node, the Calico `felix` daemon enforces your Network Policies. It programs Linux `iptables`/`IPVS` (or eBPF maps) directly at the `veth` interface, dropping unauthorized traffic at the absolute edge of the compute environment before it ever reaches the application.

# Q7 What Type of Network we used in EKS ?
  Ans:
    Actually, Amazon EKS does **not** use Calico as its default network. By default, EKS uses the **Amazon VPC CNI plugin**.

  Here is a breakdown of the type of network EKS uses by default, and why you might be seeing or hearing about Calico in EKS environments.

  ### 1. The Default: Amazon VPC CNI (Native Networking)

  Out of the box, EKS relies on native AWS networking rather than creating a virtual overlay network.

  * **Native IP Assignment:** The VPC CNI plugin assigns actual, routable IP addresses directly from your AWS VPC subnets to your Pods.
  * **How it works (ENIs):** The plugin attaches Elastic Network Interfaces (ENIs) to your EC2 worker nodes. It then grabs a "warm pool" of secondary IP addresses from           that ENI and assigns them one-by-one to the containers inside your pods.
    * **The Advantage:** Because there is no packet encapsulation (like VXLAN or IP-in-IP), your Pods get raw, bare-metal AWS network performance. They can also communicate         natively with other AWS services like RDS or Application Load Balancers.

# Q8 What is Sidecar container and INIT container and Daemon Set  based Deployment and container and difference ?

  Ans: In Kubernetes, a standard container is the foundational unit of execution, while Init and Sidecar containers are specific architectural patterns within a single Pod, and a DaemonSet is a cluster-wide strategy for placing those Pods onto nodes.

### 1. Standard Container

The primary execution unit containing your core application code and its dependencies. If a Pod has only one container, this is it.

* **Lifecycle:** Runs continuously until the application terminates or the Pod is killed.
* **Example:** A Python FastAPI backend or an Nginx web server handling user traffic.

### 2. Init Container

A specialized setup container that runs and must successfully run to completion *before* the main application containers are allowed to start. If you define multiple Init containers, they run sequentially, one after the other.

* **Lifecycle:** Starts, completes its task, and terminates. If it fails, the Pod restarts until it succeeds.
* **Example:** A script that pings a PostgreSQL database to ensure it is fully running before starting the main API container, or a script that clones a Git repository into a shared volume.

### 3. Sidecar Container

A secondary container that runs concurrently alongside the primary application container within the same Pod. Because they share the exact same network namespace (localhost) and storage volumes, the sidecar can easily enhance or intercept the main application's traffic and data.

* **Lifecycle:** Starts alongside the main container and runs continuously for the entire life of the Pod.
* **Example:** The Istio Envoy proxy (from your DO328 curriculum) which intercepts all network traffic in and out of the pod, or a Fluent Bit container that reads log files generated by the main application and ships them to Elasticsearch.

### 4. DaemonSet (Deployment Strategy)

Unlike the previous three (which are containers *inside* a Pod), a DaemonSet is a Kubernetes workload controller (like a `Deployment` or `StatefulSet`). It guarantees that exactly one copy of a specific Pod runs on every single worker node in your cluster (or a specific subset of nodes using Node Selectors).

* **Lifecycle:** Scales automatically as nodes are added or removed from the cluster.
* **Example:** The Calico CNI networking plugin (running a routing daemon on every node), a Node Exporter for Prometheus metrics, or an anti-virus agent that must monitor the underlying node OS.

---

### Key Differences Summary

| Concept | What is it? | Execution & Lifecycle | Scope |
| --- | --- | --- | --- |
| **Standard Container** | The main app. | Runs continuously. | Inside a Pod. |
| **Init Container** | A setup task. | Runs sequentially and **must terminate** before the app starts. | Inside a Pod. |
| **Sidecar Container** | A helper app. | Runs concurrently **alongside** the main app. | Inside a Pod. |
| **DaemonSet** | A workload controller. | Ensures the Pod exists as long as the Node exists. | **Cluster-wide** (Node level). |



# Q9 User Role Access Concept in K8S ?
  Ans: In Kubernetes, user access is managed through **Role-Based Access Control (RBAC)**. The RBAC system is built on answering three fundamental questions: **Who** is making the request, **What** actions they are allowed to perform, and **Where** they are allowed to perform them.

To make this work, Kubernetes uses four core API objects grouped into two categories: defining the permissions and assigning those permissions.

### 1. The Subjects (Who)

Kubernetes does not have a built-in "User" object to manage passwords or user accounts natively. It relies on external identity providers (like AWS IAM, Active Directory, or TLS certificates) to authenticate users. Once authenticated, RBAC applies to three types of subjects:

* **Users:** Actual human operators or external administrators.
* **Groups:** Collections of users (e.g., `dev-team`, `system:masters`).
* **ServiceAccounts:** Native Kubernetes identities created specifically for applications/Pods running inside the cluster to talk to the API server.

### 2. The Permissions (What)

Permissions are defined as rules that dictate which actions (**verbs**) are allowed on specific Kubernetes objects (**resources**). Standard verbs include `get`, `list`, `watch`, `create`, `update`, `patch`, and `delete`.

* **Role:** Defines permissions bounded to a **single namespace**. For example, a Role might allow an operator to `get` and `list` pods, but only in the `development` namespace.
* **ClusterRole:** Defines permissions **across the entire cluster**. This is required to manage non-namespaced resources (like Nodes or PersistentVolumes), non-resource endpoints (like `/healthz`), or to grant a specific permission across every namespace simultaneously.

### 3. The Bindings (The Connection)

A Role or ClusterRole does absolutely nothing on its own until it is attached to a Subject. This connection is called a Binding.

* **RoleBinding:** Grants the permissions defined in a Role (or ClusterRole) to a subject within a **specific namespace**.
* **ClusterRoleBinding:** Grants the permissions defined in a ClusterRole to a subject **cluster-wide**, giving them that access across every single namespace in the cluster.

### A Real-World Example

If you want to allow a developer to manage deployments exclusively in the `api-backend` namespace, you execute two steps:

1. **Define the Access:** Create a **Role** located in the `api-backend` namespace specifying the resource (`deployments`) and the permitted verbs (`create`, `update`, `delete`, `get`).
2. **Assign the Access:** Create a **RoleBinding** located in the `api-backend` namespace. Set the subject to the developer's **User** identity, and set the role reference to point to the **Role** you just created.


# Q10 Ingress and Ingresscontroller in K8S  and how its works ?
  Ans:

  In Kubernetes, managing external HTTP/HTTPS traffic requires two distinct components working together: the **Ingress** (the rules) and the **Ingress Controller** (the engine that executes the rules).

Here is the breakdown of what they are and how they work together.

### 1. Ingress (The Rules)

An `Ingress` is simply a Kubernetes API object (a YAML file). It acts as a routing manifest that defines how external HTTP/HTTPS traffic should be directed to the internal `Service` objects in your cluster.

By itself, an `Ingress` does absolutely nothing. It is just a static configuration request. It allows you to define:

* **Host-based routing:** Traffic to `api.example.com` goes to the API service, while `web.example.com` goes to the Frontend service.
* **Path-based routing:** Traffic to `[example.com/v1/](https://example.com/v1/)` goes to Service A, while `[example.com/v2/](https://example.com/v2/)` goes to Service B.
* **TLS Termination:** SSL/TLS certificates can be attached here to decrypt HTTPS traffic before passing it to the internal pods.

### 2. Ingress Controller (The Engine)

An `Ingress Controller` is the actual software application running in a Pod (typically deployed as a `Deployment` or `DaemonSet`) that reads and enforces the Ingress rules. It is a reverse proxy functioning at Layer 7 (Application Layer).

* **Common Controllers:** NGINX Ingress Controller (the most popular), Traefik, HAProxy, and cloud-specific controllers like the AWS ALB Ingress Controller.
* Unlike built-in controllers (like the Deployment or ReplicaSet controllers) that run inside the `kube-controller-manager`, Ingress Controllers are third-party components you must manually install into your cluster.

---

### How It Works: The Traffic Flow

Here is the step-by-step lifecycle of how traffic routes through this system:

1. **Deployment:** You install an Ingress Controller (e.g., NGINX) into the cluster. The controller exposes itself to the outside world, usually via a `Service` of type `LoadBalancer` or `NodePort`.
2. **Monitoring:** The Ingress Controller continuously watches the Kubernetes API server for any new, updated, or deleted `Ingress` objects across all namespaces.
3. **Dynamic Configuration:** When you apply an `Ingress` YAML file, the controller detects it. It reads the routing rules, automatically generates the underlying proxy configuration (e.g., updating the `nginx.conf` file inside the pod), and reloads the proxy without dropping existing connections.
4. **Endpoint Routing (CKA Detail):** When an external client sends a request, it hits the Ingress Controller pod. The controller evaluates the HTTP Host header and URL path against its rules. **Crucially**, the Ingress Controller usually bypasses the internal Kubernetes `Service` IP entirely; instead, it looks up the `Endpoints` of that Service and proxies the traffic directly to the target Pod IP.

*(Note for your DO328 / Istio studies: The traditional Kubernetes `Ingress` object is currently being phased out in favor of the newer **Gateway API**. When you deployed the `bookinfo-gateway` earlier, you were using this modern evolution, which splits routing responsibilities across `GatewayClass`, `Gateway`, and `HTTPRoute` resources instead of cramming everything into a single `Ingress` object).*


# Q11 What is Network Security between Pod Communication ?
  Ans: 

  By default, Kubernetes operates on a "flat network" model where every Pod can communicate with every other Pod across all namespaces without any restrictions. Securing this pod-to-pod communication requires moving to a "Zero Trust" model, which is implemented across two distinct layers: Network Policies (Layer 3/4) and a Service Mesh (Layer 7).

### 1. Network Policies (The CKA Approach)

A `NetworkPolicy` is the native Kubernetes API object used to restrict network traffic at the IP address or port level. It functions as an internal, distributed firewall.

* **How it works:** You define rules using label selectors to identify source and destination Pods or Namespaces. You can restrict both `ingress` (incoming traffic to a pod) and `egress` (outgoing traffic from a pod).
* **Default Deny:** The moment a `NetworkPolicy` selects a specific Pod, it triggers a "default deny" posture for that Pod. Any traffic not explicitly permitted by your policy rules is instantly dropped.
* **CNI Dependency:** The Kubernetes API only stores the rules; it does not enforce them. You must have a CNI plugin that supports NetworkPolicies (like Calico or Cilium) to actually program the underlying Linux `iptables` or eBPF maps to drop the packets. Basic plugins like Flannel will ignore these rules entirely.

### 2. Service Mesh & mTLS (The DO328 / Istio Approach)

While Network Policies control *whether* a connection can be made to a specific port, a Service Mesh (like OpenShift Service Mesh or Istio) secures the actual data payload and provides application-aware (Layer 7) security.

* **Mutual TLS (mTLS):** A Service Mesh injects a sidecar proxy (like Envoy) into every Pod. When Pod A communicates with Pod B, the proxies intercept the traffic. They automatically establish an mTLS tunnel, encrypting the data in transit and cryptographically verifying the identity of both workloads using certificates, completely independent of the underlying network IPs.
* **Authorization Policies:** Because the sidecar proxy understands HTTP/gRPC traffic, you can enforce highly granular rules. For example, instead of just opening port 8080, an Istio `AuthorizationPolicy` can dictate that the Frontend Pod is allowed to issue an HTTP `GET` request to the Backend Pod's `/read` path, but is explicitly blocked from sending an HTTP `POST` to the `/write` path.

### Key Differences in Pod Security

| Feature | Network Policy (Native/Calico) | Service Mesh (Istio) |
| --- | --- | --- |
| **OSI Layer** | Layer 3 & 4 (IPs, TCP/UDP Ports) | Layer 7 (HTTP, gRPC, API Paths) |
| **Traffic Encryption** | No (Traffic remains plaintext in transit) | Yes (mTLS encrypted in transit) |
| **Enforcement Mechanism** | Linux Kernel (`iptables`, IPVS, eBPF) | User Space (Envoy Sidecar Proxy) |
| **Workload Identity** | Tied to ephemeral Pod IPs and Labels | Cryptographic X.509 certificates |
