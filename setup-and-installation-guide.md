# **Setup and Installation Guide: Federated Kubernetes with EKS and GKE**

This guide provides the detailed technical steps to set up the multi-cloud federated Kubernetes environment.

---

### **Phase 1: Prerequisites and Tooling**

Ensure the following command-line tools are installed and configured on your local machine.

1.  **AWS CLI & `eksctl`**:
    *   Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) and [eksctl](https://docs.eksctl.io/introduction/installation/).
    *   Configure with `aws configure` using an IAM user with sufficient permissions to create EKS clusters and related resources.

2.  **Google Cloud SDK (`gcloud`)**:
    *   Install the [Google Cloud SDK](https://cloud.google.com/sdk/docs/install).
    *   Initialize and authenticate with `gcloud init` and `gcloud auth application-default login`. Ensure the target GCP project has the "Kubernetes Engine API" enabled.

3.  **Kubernetes Tooling (`kubectl`, Helm)**:
    *   Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/).
    *   Install [Helm v3](https://helm.sh/docs/intro/install/).

4.  **KubeFed CLI (`kubefedctl`)**:
    *   Download the appropriate binary for your OS/architecture from the [KubeFed GitHub Releases](https://github.com/kubernetes-sigs/kubefed/releases) page.
    *   Move the `kubefedctl` binary to a location in your system's PATH (e.g., `/usr/local/bin/`).

---

### **Phase 2: Cluster Provisioning**

1.  **Create AWS EKS Cluster**:
    ```bash
    eksctl create cluster \
      --name eks-federation-member \
      --region us-west-2 \
      --version 1.28 \
      --nodegroup-name standard-workers \
      --node-type t3.medium \
      --nodes 2
    ```

2.  **Create GCP GKE Cluster**:
    ```bash
    gcloud container clusters create gke-federation-host \
      --zone us-central1-a \
      --num-nodes 2 \
      --machine-type e2-medium \
      --cluster-version latest
    ```

3.  **Configure `kubectl` Contexts**:
    After creation, rename the long, auto-generated context names for easier use.
    ```bash
    # Find the full context names
    kubectl config get-contexts

    # Rename them
    kubectl config rename-context <long-eks-context-name> eks-cluster
    kubectl config rename-context <long-gke-context-name> gke-cluster
    ```

---

### **Phase 3: Install and Configure KubeFed**

1.  **Set Context to Host Cluster**: All KubeFed control plane actions are performed on the host.
    ```bash
    kubectl config use-context gke-cluster
    ```

2.  **Install KubeFed via Helm**:
    ```bash
    helm repo add kubefed https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts
    helm repo update
    kubectl create namespace kubefed-system
    helm --namespace kubefed-system install kubefed kubefed/kubefed
    ```

3.  **Join Clusters to the Federation**:
    ```bash
    # Join the host cluster to itself
    kubefedctl join gke-cluster --cluster-context gke-cluster --host-cluster-context gke-cluster --v=2

    # Join the EKS member cluster
    kubefedctl join eks-cluster --cluster-context eks-cluster --host-cluster-context gke-cluster --v=2
    ```

4.  **Verify Federation**:
    ```bash
    kubectl get kubefedclusters -n kubefed-system
    # Both clusters should show READY status as 'True'.
    ```

---

### **Phase 4: Deploy the Production Application**

Apply the federated resource manifests to the **Host Cluster** (`gke-cluster`) in the following order.

1.  **Create and Federate the Namespace**:
    ```bash
    kubectl create namespace test-ns
    kubectl apply -f kubefed-config/01-federated-namespace.yaml
    ```

2.  **Deploy the ConfigMap**:
    ```bash
    kubectl apply -f kubefed-config/02-federated-configmap.yaml
    ```

3.  **Deploy the Application Workload**:
    ```bash
    kubectl apply -f kubefed-config/03-federated-deployment.yaml
    ```

4.  **Expose the Application**:
    ```bash
    kubectl apply -f kubefed-config/04-federated-service.yaml
    ```

---

### **Phase 5: Verification**

1.  **Check Pods on GKE**:
    ```bash
    kubectl --context=gke-cluster get pods -n test-ns
    # Expect 3 pods in a 'Running' and '1/1 READY' state.
    ```

2.  **Check Pods on EKS**:
    ```bash
    kubectl --context=eks-cluster get pods -n test-ns
    # Expect 2 pods in a 'Running' and '1/1 READY' state.
    ```

3.  **Test Connectivity**:
    Retrieve the external IPs from the services in each cluster and test with `curl`.
    ```bash
    # Get GKE Load Balancer IP
    GKE_LB=$(kubectl --context=gke-cluster get svc -n test-ns prod-nginx-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    curl http://$GKE_LB

    # Get EKS Load Balancer DNS Name
    EKS_LB=$(kubectl --context=eks-cluster get svc -n test-ns prod-nginx-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
    curl http://$EKS_LB
    ```

---

### **Phase 6: Cleanup**

Run the provided `cleanup.sh` script to delete all cloud resources and avoid ongoing costs. The script deletes application resources, unjoins clusters, uninstalls KubeFed, and finally deletes the EKS and GKE clusters.
```bash
./scripts/cleanup.sh
