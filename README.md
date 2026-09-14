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
