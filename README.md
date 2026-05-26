# Cloud Platform: AWS EKS + ArgoCD + OpenTelemetry

## 1. Executive Summary

This project showcases a production-ready cloud platform built with a modern separation-of-concerns architecture. It provisions a scalable and secure Amazon Elastic Kubernetes Service (EKS) cluster using **Terraform** and deploys the [OpenTelemetry Demo](https://opentelemetry.io/docs/demo/) microservices application utilizing **GitOps** principles with **ArgoCD**.

* **Infrastructure layer:** Handled by Terraform, employing official HashiCorp AWS modules for VPC (spanning 2 Availability Zones with a single NAT Gateway to reduce cost) and EKS (v1.30, utilizing `t3.large` nodes suitable for resource-heavy observability tools).
* **Delivery layer:** Handled by ArgoCD, pulling stable upstream manifests and syncing helm-based deployments with self-healing and automated pruning enabled.

## 2. Architecture Diagram

```mermaid
flowchart LR
    subgraph Users[User Traffic]
        Internet((Internet))
    end

    subgraph GitHub[GitOps Source of Truth]
        Repo[(GitHub Repository\nTerraform & YAML)]
    end

    subgraph AWS[AWS Cloud]
        ALB[Application Load Balancer]
        
        subgraph VPC[Virtual Private Cloud]
            subgraph EKS[Amazon EKS Cluster]
                subgraph ArgoCD_NS[argocd namespace]
                    ArgoCD[ArgoCD Controller]
                end

                subgraph KubeSystem_NS[kube-system namespace]
                    LBC[AWS Load Balancer Controller]
                end

                subgraph OTel_NS[otel-demo namespace]
                    FrontendProxy[Frontend Proxy]
                    Frontend[Frontend App]
                    Backend[Backend Microservices\nCart, Payment, etc.]
                    Obs[Observability Stack\nGrafana, Jaeger, Prometheus]
                    
                    FrontendProxy --> Frontend
                    Frontend --> Backend
                    Backend -.->|Telemetry| Obs
                    Frontend -.->|Telemetry| Obs
                end
            end
        end
    end

    Internet -->|HTTP Request| ALB
    ALB -->|Ingress| FrontendProxy
    
    LBC -.->|Provisions & Manages| ALB
    ArgoCD -.->|Monitors & Syncs| Repo
    ArgoCD =.=>|Deploys| LBC
    ArgoCD =.=>|Deploys| OTel_NS

    classDef aws fill:#FF9900,stroke:#232F3E,stroke-width:2px,color:#232F3E;
    classDef k8s fill:#326CE5,stroke:#fff,stroke-width:2px,color:#fff;
    classDef git fill:#2dba4e,stroke:#fff,stroke-width:2px,color:#fff;
    classDef bg fill:#f4f4f4,stroke:#333,stroke-width:1px,stroke-dasharray: 5 5;
    
    class ALB aws;
    class ArgoCD,LBC,FrontendProxy,Frontend,Backend,Obs k8s;
    class Repo git;
    class VPC,EKS,ArgoCD_NS,KubeSystem_NS,OTel_NS bg;
```

## 3. Business Value

* **Automated Provisioning:** Reduces deployment time from days to minutes. All infrastructure is codified, minimizing configuration drift and human error.
* **GitOps Security & Reliability:** Applications are managed via declarative source control. ArgoCD constantly monitors the cluster state and automatically heals discrepancies against the desired state defined in Git.
* **Full-Stack Observability:** The OpenTelemetry Demo provides immediate, actionable insights into microservice interactions, traces, and metrics, ensuring high availability and rapid debugging for business-critical applications.

### 🏢 Architecture Best Practice: Separation of Concerns

This project strictly follows the industry-standard **Separation of Concerns** by acting as a dedicated **Platform/GitOps Repository**, completely separate from the application source code repository. 

* **Different Lifecycles:** Application code (Java, Go, Node.js) changes rapidly (multiple times a day), while Platform code (Terraform, Kubernetes) changes deliberately and slowly. Separating them prevents minor application typos from triggering massive infrastructure pipelines.
* **Security & Blast Radius:** Application developers are granted full access to merge application code in their repo, but are restricted from modifying the VPC, tearing down EKS clusters, or changing IAM permissions in this Platform repo.
* **The GitOps Standard:** When application developers finish writing code, their CI pipeline builds a new Docker image and automatically opens a Pull Request in this Platform Repository to update the image tag. ArgoCD then detects the YAML change here and securely syncs it to the cluster.

---

## 3. Deployment Guide: Prerequisites & Infrastructure (Terraform)

### Prerequisites
You must have the AWS CLI, Terraform, and `kubectl` installed on your local machine.

*If you are on a Mac, you can install them via Homebrew (Note: Terraform must be installed from HashiCorp's tap):*
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform awscli
```
Once installed, configure your AWS credentials to link your terminal to your AWS account:
```bash
aws configure
```

### Infrastructure Provisioning Steps
1. Navigate to the Terraform directory:
   ```bash
   cd terraform
   ```
2. Initialize Terraform modules and providers:
   ```bash
   terraform init
   ```
3. Apply the configuration to provision the VPC and EKS cluster:
   ```bash
   terraform apply -auto-approve
   ```
   > **Note:** This process typically takes 10-15 minutes.

4. Once complete, configure `kubectl` to communicate with your new cluster:
   ```bash
   aws eks update-kubeconfig --region us-east-1 --name otel-demo-cluster
   ```

---

## 4. Deployment Guide: Platform Delivery (GitOps)

### Install ArgoCD
Because ArgoCD's Custom Resource Definitions (CRDs) are extremely large, we use Kubernetes Server-Side Apply to bypass annotation size limits.

1. Create the `argocd` namespace:
   ```bash
   kubectl create namespace argocd
   ```
2. Apply the initial ArgoCD manifests using Server-Side Apply:
   ```bash
   kubectl apply --server-side=true --force-conflicts -k gitops/1-argocd-init
   ```

### Deploy the OpenTelemetry Demo
With ArgoCD running, apply the GitOps manifests. ArgoCD will automatically detect the Application Custom Resource, create the `otel-demo` namespace, and deploy all 25+ microservices.

```bash
kubectl apply -f gitops/2-apps/argocd-project.yaml
kubectl apply -f gitops/2-apps/opentelemetry-demo.yaml
```

![ArgoCD Dashboard](images/argocd-dashboard.png)

---

## 5. Accessing the Application & Observability Tools

Our GitOps configuration explicitly instructs AWS to provision a public Elastic Load Balancer (ELB) for the application frontend proxy.

1. Run the following command and wait for the `EXTERNAL-IP` to populate with an AWS domain name (it usually takes 2-3 minutes):
   ```bash
   kubectl get svc frontend-proxy -n otel-demo -w
   ```
2. Copy the Load Balancer URL. You can access the different components of the platform by appending the correct paths to your URL.

* **Astronomy Shop Frontend:** `http://<YOUR_AWS_ELB_URL>:8080/`
* **Grafana Dashboards:** `http://<YOUR_AWS_ELB_URL>:8080/grafana/`

![Grafana RED Metrics](images/grafana-red.png)
![Grafana Cart Exemplars](images/grafana-cart.png)

* **Jaeger Distributed Tracing:** `http://<YOUR_AWS_ELB_URL>:8080/jaeger/ui/`

![Jaeger Distributed Tracing](images/jaeger-tracing.png)

---

## 6. Continuous Integration (GitHub Actions)

This repository includes a GitHub Actions CI pipeline (`.github/workflows/ci.yml`) that triggers on every push or pull request to the `main` branch. 

![GitHub Actions Pipeline](images/github-actions.png)

The pipeline ensures code quality and safety by running:
1. **Terraform Validation:** Enforces formatting (`terraform fmt`) and validates the AWS infrastructure code (`terraform validate`).
2. **YAML Linting:** Scans the `gitops/` directory with `yamllint` to ensure all Kubernetes manifests are syntactically valid.

---

## 7. Comprehensive Deployment Journey

For a detailed step-by-step walkthrough of exactly how this architecture was built, including the real-world roadblocks encountered (like macOS dependency issues, ArgoCD CRD size limits, and local port conflicts) and how they were solved, please see the [Deployment Journey & Troubleshooting Guide](deployment_journey.md).

---

## 8. Upgrading to Application Load Balancer (ALB)

To optimize costs and enable advanced Layer 7 routing, the platform uses the **AWS Load Balancer Controller** instead of basic Classic Load Balancers.

1. **IAM Role (Terraform):** An IAM Role for Service Accounts (IRSA) was provisioned in Terraform and linked to the EKS OIDC provider.
2. **ALB Controller (GitOps):** An ArgoCD application (`gitops/2-apps/aws-lbc.yaml`) automatically deploys the controller into the `kube-system` namespace.
3. **Ingress Migration (GitOps):** The OpenTelemetry Demo is configured to use a Kubernetes `Ingress` resource with the `alb` ingress class, instructing AWS to provision a modern Application Load Balancer.

> [!NOTE]
> **Strict Helm Schemas in GitOps:** During the migration, the OpenTelemetry v2.2.0 Helm chart rejected the initial configuration due to a strict JSON Schema validation (the component key changed from `frontendProxy` to `frontend-proxy`). Thanks to ArgoCD, the deployment failed gracefully without taking down the existing ELB. Updating the manifest to strictly match the new schema allowed ArgoCD to provision the ALB perfectly.

---

## 9. Cost Warning & Teardown (CRITICAL)

> [!WARNING]
> **AWS charges apply for resources running in this project.** An EKS control plane, `t3.large` worker nodes, a NAT Gateway, and an ELB are not part of the AWS Free Tier. They cost approximately **$0.35/hour** ($250/month).

Ensure you destroy all resources when you are finished to prevent unexpected billing.

1. **Delete the ArgoCD Applications First:**
   ArgoCD application finalizers can block namespace deletion, and external resources (like the Application Load Balancer) must be gracefully deleted by the cluster before the cluster itself is destroyed.
   ```bash
   kubectl delete -f gitops/2-apps/opentelemetry-demo.yaml
   kubectl delete -f gitops/2-apps/aws-lbc.yaml
   ```
   *(Wait ~3 minutes for the Load Balancer to fully delete in AWS before proceeding)*

2. **Destroy Infrastructure:**
   Navigate back to the terraform directory and issue the destroy command:
   ```bash
   cd terraform
   terraform destroy -auto-approve
   ```
