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
