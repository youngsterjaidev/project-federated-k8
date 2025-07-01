# **Project Report: Multi-Cloud Federated Kubernetes with AWS EKS and GCP GKE**

**Author:** JAI DEV
**Version:** 1.0

---

### **1. Executive Summary**

This project successfully established a multi-cloud, federated Kubernetes cluster spanning Amazon Web Services (AWS) and Google Cloud Platform (GCP). Using an AWS Elastic Kubernetes Service (EKS) cluster and a Google Kubernetes Engine (GKE) cluster, a unified control plane was created with **KubeFed (Kubernetes Federation v2)**. This architecture enables seamless deployment and management of containerized applications across both clouds from a single point of interaction. The primary objectives of enhancing high availability, providing a robust disaster recovery strategy, and enabling flexible resource utilization were met and validated through the deployment and testing of a production-grade NGINX application. This federated model mitigates vendor lock-in and allows the organization to leverage the distinct advantages of each cloud provider.

---

### **2. Introduction**

#### **2.1. Problem Statement**

In the modern cloud landscape, exclusive reliance on a single cloud provider introduces significant business risks, including vendor lock-in, vulnerability to large-scale outages, and potentially suboptimal cost structures. The alternative, managing applications across multiple clouds, traditionally involves siloed operational tools and duplicated deployment efforts, which increases complexity and reduces engineering velocity.

#### **2.2. Project Objectives**

The primary goal of this project was to overcome these challenges by achieving the following:

*   **Unified Management:** To deploy, manage, and update a containerized workload across both AWS and GCP from a single point of control.
*   **High Availability (HA):** To ensure continuous application uptime by running active workloads simultaneously in physically and provider-isolated environments.
*   **Disaster Recovery (DR):** To build an architecture capable of withstanding a catastrophic failure of an entire cloud region.
*   **Resource Optimization:** To enable intelligent workload scheduling and scaling based on the unique costs, capabilities, or performance of each cloud provider.

---

### **3. Architecture and Technology Stack**

#### **3.1. Technology Stack**

*   **Cloud Platforms:** Amazon Web Services (AWS), Google Cloud Platform (GCP)
*   **Kubernetes Services:** AWS Elastic Kubernetes Service (EKS), Google Kubernetes Engine (GKE)
*   **Federation Layer:** KubeFed (Kubernetes Federation v2)
*   **Container Image:** NGINX (`nginx:1.25`)
*   **Infrastructure & Deployment Tools:** `kubectl`, `helm`, `eksctl`, `gcloud`, `aws-cli`, `kubefedctl`

#### **3.2. Architecture Overview**

The solution is based on the KubeFed Host/Member model, which provides a logical separation of concerns:

*   **Host Cluster:** The **GCP GKE cluster** was designated as the host. It runs the KubeFed control plane, which houses the federated API server and controllers. All user interactions for managing federated resources occur here.
*   **Member Clusters:** Both the **GCP GKE cluster** and the **AWS EKS cluster** were joined to the federation as members. These clusters are responsible for running the actual application workloads as dictated by the host cluster.

The control flow is as follows: A developer applies a federated manifest (e.g., `FederatedDeployment`) to the Host Cluster. The KubeFed controller detects this, processes the placement and override rules, and then creates corresponding native Kubernetes resources (e.g., `Deployment`, `Service`) on the designated Member Clusters.

---

### **4. Validation and Testing**

*   **Deployment Verification:** `kubectl get pods -n test-ns` was run against both the `eks-cluster` and `gke-cluster` contexts, confirming the correct number of pods (2 on EKS, 3 on GKE) were `Running` with a `READY 1/1` status.
*   **Connectivity Test:** The external `LoadBalancer` endpoints for both EKS and GKE were successfully tested using `curl`, confirming end-to-end network connectivity.
*   **Self-Healing Test:** A pod failure was simulated by manual deletion. The Kubernetes `Deployment` controller on the member cluster automatically and immediately provisioned a replacement pod, successfully demonstrating the system's high-availability features.
*   **Debugging:** Real-world issues were encountered and resolved, including:
    *   **`CrashLoopBackOff`:** Caused by a `securityContext` permissions conflict, resolved by providing `emptyDir` volumes for temporary data.
    *   **`NamespaceNotFederated`:** Resolved by ensuring the application namespace was itself created via a `FederatedNamespace` resource.
    *   **Label/Selector Mismatch:** Resolved by correcting pod labels and service selectors for proper service endpoint discovery.

---

### **5. Conclusion and Future Work**

This project successfully demonstrates a powerful, modern approach to multi-cloud infrastructure management. The federated Kubernetes architecture provides a tangible solution to the challenges of vendor lock-in and high availability.

**Future Work:**

*   **Automated Failover:** Implement a global DNS load balancer (e.g., AWS Route 53) to automate traffic redirection in a disaster recovery scenario.
*   **Federated Observability:** Integrate a federated monitoring solution like Prometheus with Thanos or Grafana Mimir to create a single pane of glass for multi-cluster metrics.
*   **CI/CD Automation:** Build a CI/CD pipeline (e.g., using GitLab CI or Jenkins) to automate the testing and deployment of federated applications.
