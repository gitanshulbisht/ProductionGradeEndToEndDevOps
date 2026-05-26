# Platform Deployment Journey & Troubleshooting Guide

This document captures the entire end-to-end journey of provisioning the AWS EKS cluster, deploying ArgoCD, and running the OpenTelemetry Demo application. It serves as a practical guide for anyone recreating this GitOps-driven cloud platform, highlighting both the successes and the real-world roadblocks we encountered (and solved).

---

## 1. Project Architecture
* **Infrastructure:** AWS VPC and Amazon EKS (Kubernetes 1.30) provisioned via **Terraform**.
* **Delivery:** **ArgoCD** managing applications via GitOps.
* **Application:** The official **OpenTelemetry Demo**, deployed via Helm through ArgoCD.

---

## 2. Infrastructure Provisioning (Terraform)

### The Setup
We generated the infrastructure code using official HashiCorp AWS modules (`terraform-aws-modules/vpc/aws` and `terraform-aws-modules/eks/aws`), placing it in a `/terraform` directory.

### 🛑 Issue: Missing Dependencies & Licensing Changes
When attempting to execute Terraform, the local machine was missing both the AWS CLI and Terraform. 
* **What didn't work:** Running the standard `brew install terraform` failed. HashiCorp recently changed their software licensing, so Homebrew removed Terraform from their core repositories.
* **The Fix:** We installed Terraform directly from HashiCorp's official Homebrew tap alongside the AWS CLI:
  ```bash
  brew tap hashicorp/tap
  brew install hashicorp/tap/terraform awscli
  ```

### Execution
After the user configured their AWS credentials (`aws configure`), we ran:
```bash
terraform init
terraform apply -auto-approve
```
**Result:** Worked flawlessly. The VPC, NAT Gateways, and EKS Cluster (`otel-demo-cluster` with `t3.large` nodes) were provisioned in roughly 10 minutes.

---

## 3. Platform Delivery (ArgoCD & GitOps)

### The Setup
We configured our local Kubernetes context using the AWS CLI:
```bash
aws eks update-kubeconfig --region us-east-1 --name otel-demo-cluster
```

### 🛑 Issue: Kubernetes CRD Annotation Size Limits
We attempted to install ArgoCD using standard client-side apply (`kubectl apply -k gitops/1-argocd-init`).
* **What didn't work:** The command failed at the very end with: `The CustomResourceDefinition "applicationsets.argoproj.io" is invalid: metadata.annotations: Too long: must have at most 262144 bytes.` ArgoCD's Custom Resource Definitions are so massive that they exceed Kubernetes' standard annotation limits.
* **The Fix:** We switched to Kubernetes **Server-Side Apply** and forced conflicts to overwrite the partially applied resources:
  ```bash
  kubectl apply --server-side=true --force-conflicts -k gitops/1-argocd-init
  ```

### Execution
Once ArgoCD was running, we applied the GitOps manifests:
```bash
kubectl apply -f gitops/2-apps/argocd-project.yaml
kubectl apply -f gitops/2-apps/opentelemetry-demo.yaml
```
**Result:** ArgoCD successfully detected the Application Custom Resource, created the `otel-demo` namespace, and pulled down the 25+ OpenTelemetry microservices via Helm.

---

## 4. Accessing the Application Locally

### 🛑 Issue: Local Port Conflicts & Nginx
To view the application securely without a public IP, we attempted to port-forward the traffic to `localhost:8080`. 
* **What didn't work:** The user saw a "Welcome to Nginx" page instead of the Astronomy Shop. The OpenTelemetry frontend uses Next.js, meaning the user had a local Docker container or Nginx server already hijacking port 8080 on their Mac.
* **The Fix:** We shifted the local port to `8088` and targeted the correct service name (`frontend-proxy`):
  ```bash
  kubectl port-forward svc/frontend-proxy 8088:8080 -n otel-demo
  ```

---

## 5. Exposing the Application Globally (AWS ELB)

The user wanted the application to be accessible globally over the internet without requiring terminal port-forwarding.

### 🛑 Issue: Default ClusterIP Service
By default, the OpenTelemetry Helm chart deploys the `frontend-proxy` as an internal `ClusterIP` service, keeping it hidden from the internet.

### The Fix: Overriding Helm Values via GitOps
We updated the ArgoCD Application manifest (`gitops/2-apps/opentelemetry-demo.yaml`) to inject custom Helm values. We changed the proxy service type to `LoadBalancer`:

```yaml
    helm:
      values: |
        components:
          frontendProxy:
            service:
              type: LoadBalancer
```
We applied the manifest (`kubectl apply -f gitops/2-apps/opentelemetry-demo.yaml`). ArgoCD instantly detected the change, updated the Kubernetes Service, and AWS automatically provisioned a public Elastic Load Balancer (ELB). 

**Result:** The application was immediately available globally at the AWS ELB URL (e.g., `http://xxx.us-east-1.elb.amazonaws.com:8080`).

---

## 6. Accessing Observability Tools (Jaeger & Grafana)

The OpenTelemetry Demo includes full-stack tracing and metrics. 
* **What worked beautifully:** We did **not** need to provision extra Load Balancers for Grafana or Jaeger (which saved AWS costs). The OpenTelemetry Envoy `frontend-proxy` is natively configured to route traffic based on URL paths. 

Using the exact same public AWS Load Balancer URL, the observability tools were immediately accessible:
* **Astronomy Shop:** `http://<YOUR_AWS_ELB_URL>:8080/`
* **Grafana Dashboards:** `http://<YOUR_AWS_ELB_URL>:8080/grafana/`
* **Jaeger Distributed Tracing:** `http://<YOUR_AWS_ELB_URL>:8080/jaeger/ui/`

---

## 7. Cost Warning & Cleanup

> [!CAUTION]
> **This architecture is not covered by the AWS Free Tier.** Running the EKS Control Plane, the `t3.large` nodes, the NAT Gateway, and the ELB costs approximately **$0.35 per hour** ($250/month). 

To cleanly destroy this platform and prevent billing:
1. First, delete the GitOps application to clean up ELBs and namespaces:
   ```bash
   kubectl delete -f gitops/2-apps/opentelemetry-demo.yaml
   ```
2. Destroy the Terraform infrastructure:
   ```bash
   cd terraform
   terraform destroy -auto-approve
   ```

---

## 8. Continuous Integration (GitHub Actions)

To ensure that future changes to the infrastructure and GitOps manifests are safe, we implemented a CI pipeline using GitHub Actions.

### The Pipeline (`.github/workflows/ci.yml`)
The workflow automatically triggers on every push and pull request to the `main` branch. It consists of two parallel jobs:

1. **Terraform Validation (`terraform-validate`):**
   * Checks out the code and sets up the HashiCorp Terraform CLI.
   * Runs `terraform fmt -check` to enforce proper HCL formatting.
   * Runs `terraform init -backend=false` and `terraform validate` to catch any syntactical or logical errors in the AWS infrastructure definition without needing actual AWS credentials.

2. **YAML Linting (`yaml-lint`):**
   * Installs and runs `yamllint` on the `gitops/` directory.
   * Uses a custom `.yamllint` configuration file to ignore Kubernetes-specific long line limits (like base64 certificates or huge CRDs).

**Result:** The repository is now protected from unformatted or broken infrastructure code being merged.
