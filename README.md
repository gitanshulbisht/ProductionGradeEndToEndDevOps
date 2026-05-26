# Cloud Platform: AWS EKS + ArgoCD + OpenTelemetry

## 1. Executive Summary

This project showcases a production-ready cloud platform built with a modern separation-of-concerns architecture. It provisions a scalable and secure Amazon Elastic Kubernetes Service (EKS) cluster using **Terraform** and deploys the [OpenTelemetry Demo](https://opentelemetry.io/docs/demo/) microservices application utilizing **GitOps** principles with **ArgoCD**.

* **Infrastructure layer:** Handled by Terraform, employing official HashiCorp AWS modules for VPC (spanning 2 Availability Zones with a single NAT Gateway to reduce cost) and EKS (v1.30, utilizing `t3.large` nodes suitable for resource-heavy observability tools).
* **Delivery layer:** Handled by ArgoCD, pulling stable upstream manifests and syncing helm-based deployments with self-healing and automated pruning enabled.

## 2. Business Value

* **Automated Provisioning:** Reduces deployment time from days to minutes. All infrastructure is codified, minimizing configuration drift and human error.
* **GitOps Security & Reliability:** Applications are managed via declarative source control. ArgoCD constantly monitors the cluster state and automatically heals discrepancies against the desired state defined in Git.
* **Full-Stack Observability:** The OpenTelemetry Demo provides immediate, actionable insights into microservice interactions, traces, and metrics, ensuring high availability and rapid debugging for business-critical applications.
* **Cost Efficiency:** Using a single NAT Gateway and appropriately sized `t3.large` EC2 spot/on-demand nodes strikes a balance between performance and expenditure.

---

## 3. Deployment Guide: Infrastructure (Terraform)

### Prerequisites
* AWS CLI installed and configured (`aws configure`).
* Terraform installed (v1.3.0+).
* `kubectl` installed.

### Steps
1. Navigate to the Terraform directory:
   ```bash
   cd terraform
   ```
2. Initialize Terraform modules and providers:
   ```bash
   terraform init
   ```
3. (Optional) Customize variables. Copy the example file and modify as needed:
   ```bash
   cp terraform.tfvars.example terraform.tfvars
   ```
4. Review the infrastructure plan:
   ```bash
   terraform plan
   ```
5. Apply the configuration to provision the VPC and EKS cluster:
   ```bash
   terraform apply -auto-approve
   ```
   > **Note:** This process typically takes 15-20 minutes.

6. Configure `kubectl` using the output command from Terraform:
   ```bash
   $(terraform output -raw configure_kubectl)
   ```
   Verify cluster access:
   ```bash
   kubectl get nodes
   ```

---

## 4. Deployment Guide: Platform Delivery (GitOps)

### Install ArgoCD
ArgoCD will manage the deployment of our applications.
1. Create the `argocd` namespace:
   ```bash
   kubectl create namespace argocd
   ```
2. Apply the initial ArgoCD manifests:
   ```bash
   kubectl apply -k gitops/1-argocd-init
   ```
3. Wait for the ArgoCD server to be ready:
   ```bash
   kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s
   ```

### Deploy the OpenTelemetry Demo
With ArgoCD running, apply the GitOps manifests to deploy the OpenTelemetry Application.
1. Apply the ArgoCD Project and Application manifests:
   ```bash
   kubectl apply -f gitops/2-apps/argocd-project.yaml
   kubectl apply -f gitops/2-apps/opentelemetry-demo.yaml
   ```
2. ArgoCD will automatically detect the Application Custom Resource, create the `otel-demo` namespace, and begin syncing the Helm chart.
   
3. Monitor the deployment progress:
   ```bash
   kubectl get pods -n otel-demo -w
   ```

---

## 5. Accessing the Application

Once all pods in the `otel-demo` namespace are in the `Running` state, port-forward the OpenTelemetry Demo frontend to view the application on your local machine.

```bash
kubectl port-forward svc/opentelemetry-demo-frontend 8080:8080 -n otel-demo
```
Open your web browser and navigate to: [http://localhost:8080](http://localhost:8080)

*The OpenTelemetry UI (Jaeger/Grafana) can also be accessed via their respective services in the same namespace depending on the Helm chart defaults.*

---

## 6. Cost Warning & Teardown (CRITICAL)

> [!WARNING]
> **AWS charges apply for resources running in this project.** An EKS control plane and `t3.large` worker nodes are not part of the AWS Free Tier. You will incur charges for the time they are active.

When you are finished testing the demo, ensure you destroy all resources to prevent unexpected billing.

1. **Delete the ArgoCD Application First (Important):**
   ArgoCD application finalizers can block namespace deletion. Delete the application first to properly clean up Kubernetes resources:
   ```bash
   kubectl delete -f gitops/2-apps/opentelemetry-demo.yaml
   ```
2. **Destroy Infrastructure:**
   Navigate back to the terraform directory and issue the destroy command:
   ```bash
   cd terraform
   terraform destroy -auto-approve
   ```
3. Verify that the VPC and EKS cluster have been successfully terminated in your AWS Console.
