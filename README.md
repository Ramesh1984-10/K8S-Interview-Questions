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
