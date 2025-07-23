# Home Lab Setup
Setup of Home Lab inside Oracle Cloud Free Tier

Comprehensive DevOps Home Lab Setup Guide on Oracle Cloud Free Tier
===================================================================

This guide provides a detailed, step-by-step process to set up a cloud-based DevOps home lab on Oracle Cloud's Always Free tier, tailored to showcase DevOps skills in Terraform, Ansible, Docker, and Kubernetes. Designed for a low-spec machine (4+ year-old i7, 16GB RAM, 250GB SSD), the setup leverages Oracle's Cloud Shell to avoid local resource constraints. The lab is complex, yet simple enough to deploy in 15-25 minutes (excluding background provisioning). It includes networking (Virtual Cloud Networks, load balancers) and storage (block and object storage).

Prerequisites
-------------

-   **Oracle Cloud Account**: Sign up at [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/). A credit card and mobile number are required for verification, but no charges apply for Always Free resources.
-   **Basic Knowledge**: Familiarity with Terraform, Ansible, and Kubernetes is helpful but not mandatory, as this guide provides detailed instructions.
-   **No Local Installation**: Oracle's Cloud Shell provides pre-installed tools (Terraform, Ansible, OCI CLI, kubectl, Git).
-   **Time Estimate**: 15-25 minutes for active setup, with 10-15 minutes additional for cluster provisioning (runs in the background).

Skills Demonstrated
-------------------

This lab covers key DevOps skills :

-   **Infrastructure as Code (IaC)**: Provisioning a Kubernetes cluster with Terraform.
-   **Configuration Management**: Deploying applications with Ansible.
-   **Containerization and Orchestration**: Running Docker containers in a Kubernetes cluster.
-   **Cloud Platforms**: Managing Oracle Cloud Infrastructure (OCI) resources.
-   **Networking**: Configuring VCNs, subnets, and load balancers.
-   **Storage**: Utilizing block and object storage for persistent data.
-   **CI/CD (Optional)**: Setting up GitHub Actions for automation.
-   **Monitoring (Optional)**: Adding Prometheus and Grafana for observability.

Step-by-Step Setup Instructions
-------------------------------

### Step 1: Sign Up for Oracle Cloud Free Tier

1.  **Create an Account**:
    -   Visit [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/) and click **Start for Free**.
    -   Provide your email, mobile number, and credit card for verification (no charges for Always Free resources).
    -   Choose a **home region** carefully (e.g., `us-phoenix-1`), as Always Free resources are region-specific. Avoid `us-ashburn-1` if capacity issues arise.
2.  **Verify Account**:
    -   Complete email and phone verification.
    -   Log in to the OCI Console after account activation.

**Estimated Time**: 5-10 minutes (depending on verification speed).

### Step 2: Upgrade to Pay As You Go

1.  **Access Billing**:
    -   In the OCI Console, navigate to **Billing and Cost Management** > **Upgrade and Manage Payment**.
2.  **Upgrade Account**:
    -   Select **Pay As You Go** to unlock Oracle Kubernetes Engine (OKE) access.
    -   Ensure you only use Always Free resources (e.g., VM.Standard.A1.Flex instances) to avoid charges.
3.  **Note Compartment OCID**:
    -   Go to **Identity** > **Compartments** and copy the OCID of your root compartment (or create a new one).

**Why Upgrade?**: OKE requires a Pay As You Go account, but you can configure it to use only free-tier resources.

**Estimated Time**: 2-3 minutes.

### Step 3: Access Cloud Shell

1.  **Open Cloud Shell**:
    -   In the OCI Console, click the Cloud Shell icon (terminal symbol) at the top right.
    -   This provides a Linux environment with pre-installed tools: Terraform, Ansible, OCI CLI, kubectl, and Git.
2.  **Verify Tools**:

    ```
    terraform --version
    ansible --version
    kubectl version --client

    ```

    -   Confirm versions are displayed, ensuring tools are ready.

**Estimated Time**: 2 minutes.

### Step 4: Provision OKE Cluster with Terraform

1.  **Create Project Directory**:

    ```
    mkdir oke-cluster && cd oke-cluster

    ```

2.  **Create Terraform Configuration**:
    -   Create a file named `main.tf` with the following content, replacing placeholders with your values:

        ```
        provider "oci" {
          tenancy_ocid     = "ocid1.tenancy.oc1..example"
          user_ocid        = "ocid1.user.oc1..example"
          fingerprint      = "your_fingerprint"
          private_key_path = "~/.oci/oci_api_key.pem"
          region           = "us-phoenix-1"  # Replace with your region
        }

        module "oke" {
          source  = "oracle-terraform-modules/oke/oci"
          version = "5.2.2"

          compartment_id = "ocid1.compartment.oc1..example"  # Your compartment OCID
          vcn_cidr       = "10.0.0.0/16"
          cluster_name   = "my-oke-cluster"
          kubernetes_version = "v1.28.2"  # Check latest version in OCI Console
          node_pool_shape = "VM.Standard.A1.Flex"  # Always Free eligible
          node_pool_size  = 2
          node_pool_ocpus = 2
          node_pool_memory = 12
        }

        ```

3.  **Obtain Credentials**:
    -   Navigate to **Identity** > **Users** > [Your User] > **API Keys** in the OCI Console.
    -   Generate an API key pair, download the private key, and upload it to Cloud Shell at `~/.oci/oci_api_key.pem`.
    -   Copy `tenancy_ocid`, `user_ocid`, and `fingerprint` from the API Keys section.
4.  **Initialize and Apply Terraform**:

    ```
    terraform init
    terraform plan
    terraform apply

    ```

    -   Review the plan and type `yes` to approve. This provisions:
        -   An OKE cluster with two worker nodes (VM.Standard.A1.Flex, 2 OCPUs, 12GB RAM each).
        -   A Virtual Cloud Network (VCN) with public/private subnets and a load balancer.
    -   **Note**: Cluster provisioning may take 10-15 minutes but runs in the background.

**Estimated Time**: 5-7 minutes (excluding provisioning).

### Step 5: Configure Kubectl

1.  **Obtain Kubeconfig**:
    -   Find the `cluster_ocid` in the OCI Console under **Developer Services** > **Kubernetes Clusters (OKE)**.
    -   Run:

        ```
        oci ce cluster create-kubeconfig --cluster-id <cluster_ocid> --file $HOME/.kube/config

        ```

2.  **Verify Cluster**:

    ```
    kubectl get nodes

    ```

    -   This should list two worker nodes, confirming the cluster is active.

**Estimated Time**: 2-3 minutes.

### Step 6: Deploy Application with Ansible

1.  **Create Ansible Playbook**:
    -   Create a file named `deploy.yml`:

        ```
        - name: Deploy NGINX
          hosts: localhost
          tasks:
            - name: Create deployment
              k8s:
                state: present
                definition:
                  apiVersion: apps/v1
                  kind: Deployment
                  metadata:
                    name: nginx-deployment
                  spec:
                    replicas: 1
                    selector:
                      matchLabels:
                        app: nginx
                    template:
                      metadata:
                        labels:
                          app: nginx
                      spec:
                        containers:
                        - name: nginx
                          image: nginx:1.14.2
                          ports:
                          - containerPort: 80
            - name: Create service
              k8s:
                state: present
                definition:
                  apiVersion: v1
                  kind: Service
                  metadata:
                    name: nginx-service
                  spec:
                    selector:
                      app: nginx
                    ports:
                      - protocol: TCP
                        port: 80
                        targetPort: 80
                    type: LoadBalancer

        ```

2.  **Run Playbook**:

    ```
    ansible-playbook deploy.yml

    ```

3.  **Access Application**:

    ```
    kubectl get svc nginx-service

    ```

    -   Note the external IP of the load balancer (may take a few minutes to provision).
    -   Open the IP in a browser to see the NGINX welcome page.

**Estimated Time**: 5-6 minutes.


Networking and Storage Details
------------------------------

-   **Networking**:
    -   The Terraform OKE module creates:
        -   A Virtual Cloud Network (VCN) with a CIDR block (e.g., `10.0.0.0/16`).
        -   Public and private subnets for secure communication.
        -   A Flexible Load Balancer (10 Mbps, free tier) to expose the NGINX service.
    -   This setup demonstrates your ability to manage cloud networking, a key DevOps skill.
-   **Storage**:
    -   **Block Storage**: 200 GB, attachable to nodes for Kubernetes PersistentVolumes.
    -   **Object Storage**: 20 GB, suitable for backups or static files.
    -   Example: Create a block volume in the OCI Console and configure it as a PersistentVolume (requires additional setup).

Oracle Cloud Free Tier Storage
------------------------------

-   **Lifetime Free Storage**:
    -   **Block Volume Storage**: 200 GB, ideal for persistent data in Kubernetes.
    -   **Object Storage**: 20 GB, for storing static files or backups.
-   These resources are available indefinitely, making your local storage requirement unnecessary.

Why Oracle Cloud is the Best Provider
-------------------------------------

Based on your requirements (free tier, Kubernetes support, low-spec machine compatibility), Oracle Cloud stands out:

-   **Generous Free Tier**: Provides 4 OCPUs, 24GB RAM, 200 GB block storage, and 20 GB object storage, sufficient for a Kubernetes cluster.
-   **OKE Support**: Allows managed Kubernetes setup within free-tier limits.
-   **Cloud Shell**: Eliminates local installation needs, perfect for your low-spec laptop.
-   **Networking and Storage**: Includes VCNs, load balancers, and ample storage, aligning with DevOps job requirements.

**Comparison with Other Providers**:

| **Provider** | **Free Tier Details** | **Suitability for DevOps Lab** |
| --- | --- | --- |
| **Oracle Cloud** | 200 GB block storage, 20 GB object storage, OKE | Best: Generous resources, OKE support, Cloud Shell for low-spec machines. |
| **AWS** | 750 hours EC2, 30 GB S3 (1 year) | Limited: EKS setup exceeds free tier; local tool installation may strain your laptop. |
| **Google Cloud** | $300 credit (90 days) | Moderate: GKE possible during trial, but costs post-trial; requires local setup. |
| **Azure** | $200 credit (30 days) | Moderate: AKS may exceed free limits; local setup challenging for low-spec machines. |
| **DigitalOcean** | $100 credit (60 days) | Limited: No perpetual free tier; Kubernetes setup may incur costs. |

Optional Enhancements
---------------------

-   **CI/CD Pipeline**:
    -   Create a GitHub repository with a simple web app.
    -   Configure GitHub Actions to build Docker images and deploy to OKE.
-   **Monitoring**:
    -   Install Prometheus and Grafana using Helm charts for cluster monitoring.
    -   Example:

        ```
        helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
        helm install prometheus prometheus-community/prometheus

        ```

-   **Persistent Storage**:
    -   Create a block volume in the OCI Console and configure a PersistentVolumeClaim in Kubernetes.

DevOps Skills Demonstrated
----------------------------

| **Skill** | **How Demonstrated** |
| --- | --- |
| Infrastructure as Code (IaC) | Provisioning OKE cluster and networking with Terraform. |
| Configuration Management | Deploying NGINX using Ansible's k8s module. |
| Containerization | Running Docker containers (NGINX) in Kubernetes pods. |
| Orchestration | Managing applications on an OKE cluster. |
| Cloud Platforms | Using Oracle Cloud Infrastructure's Always Free tier. |
| Networking | Configuring VCNs, subnets, and load balancers. |
| Storage | Utilizing block and object storage for persistent data. |
| CI/CD (Optional) | Automating deployments with GitHub Actions. |
| Monitoring (Optional) | Installing Prometheus and Grafana for observability. |

Notes and Considerations
------------------------

-   **Time Constraint**: Active setup takes ~15-25 minutes, but cluster provisioning may add 10-15 minutes (background process). Start Terraform early and prepare Ansible playbooks concurrently.
-   **Resource Limits**: Stay within free-tier limits (4 OCPUs, 24GB RAM total). Monitor usage in the OCI Console under **Governance** > **Limits, Quotas and Usage**.
-   **Capacity Issues**: If you encounter an "out of host capacity" error, try a different availability domain or region, or retry later.


Resources
---------

-   [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/)
-   [OCI Free Tier Documentation](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier.htm)
-   [OKE Free Tier Tutorial](https://www.reddit.com/r/selfhosted/comments/15sy5cl/kubernetes_on_oracle_cloudfor_free/)
-   [Terraform OKE Module](https://github.com/oracle-terraform-modules/terraform-oci-oke)
-   [Ansible Kubernetes Module](https://docs.ansible.com/ansible/latest/collections/kubernetes/core/k8s_module.html)
