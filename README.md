# Amazon EKS Microservices GitOps with Argo CD

## Overview

This project demonstrates the deployment of a **cloud-native microservices application on Amazon EKS using Kubernetes and GitOps principles**.

The project uses **GitHub, GitHub Actions, Docker, Docker Hub, Kubernetes, Amazon EKS, and Argo CD** to implement an automated CI/CD and GitOps workflow.

GitHub acts as the source of truth for both application source code and Kubernetes desired-state configuration. **GitHub Actions** performs the CI process by running security checks, building container images, scanning them for vulnerabilities, and pushing them to Docker Hub.

For continuous delivery, **Argo CD** monitors the Kubernetes manifests stored in Git and continuously reconciles the desired state with the actual state running in Amazon EKS.

An **Argo CD ApplicationSet with a Git Directory Generator** is used to automatically generate individual Argo CD Applications for the microservices, eliminating the need to manually create an Application manifest for every service.

---

## Architecture

```text
                              Developer
                                  │
                                  │ git push
                                  ▼
                         ┌──────────────────┐
                         │      GitHub      │
                         │                  │
                         │ Source Code      │
                         │ K8s Manifests    │
                         └────────┬─────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
          ┌──────────────────┐         ┌──────────────────┐
          │  GitHub Actions  │         │     Argo CD      │
          │                  │         │                  │
          │ Gitleaks         │         │  ApplicationSet  │
          │ Checkov          │         │  Git Generator   │
          │ Docker Build     │         └────────┬─────────┘
          │ Trivy            │                  │
          │ Docker Push      │                  │
          └────────┬─────────┘                  │
                   │                            │
                   ▼                            │
          ┌──────────────────┐                  │
          │    Docker Hub    │                  │
          │                  │                  │
          │ Container Images │                  │
          └────────┬─────────┘                  │
                   │                            │
                   │                            │
                   └──────────────┬─────────────┘
                                  │
                                  ▼
                         ┌──────────────────┐
                         │    Amazon EKS    │
                         │                  │
                         │    Kubernetes    │
                         │     Cluster      │
                         └────────┬─────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
             Microservices              Supporting Services
                    │
                    ▼
              Application
               Workloads
```

---

## Technology Stack

| Category                | Technologies           |
| ----------------------- | ---------------------- |
| Cloud                   | Amazon Web Services    |
| Container Orchestration | Amazon EKS, Kubernetes |
| CI/CD                   | GitHub Actions         |
| Containerization        | Docker                 |
| Container Registry      | Docker Hub             |
| GitOps                  | Argo CD                |
| Application Management  | Argo CD ApplicationSet |
| Secret Scanning         | Gitleaks               |
| Kubernetes Security     | Checkov                |
| Container Security      | Trivy                  |
| Source Control          | GitHub                 |

---

## Repository Structure

The repository separates application source code from Kubernetes deployment configuration and GitOps configuration.

```text
eks-microservices-gitops/
│
├── src/
│   ├── accounting/
│   ├── add/
│   ├── cart/
│   ├── checkout/
│   ├── currency/
│   ├── email/
│   ├── flagd-ui/
│   ├── flagd/
│   ├── fraud-detection/
│   ├── frontend-proxy/
│   ├── frontend/
│   ├── grafana/
│   ├── image-provider/
│   ├── jaeger/
│   ├── kafka/
│   ├── load-generator/
│   ├── opensearch/
│   ├── otel-collector/
│   ├── payment/
│   ├── postgres/
│   ├── product-catalog/
│   ├── prometheus/
│   ├── quote/
│   ├── react-native-app/
│   ├── recommendation/
│   └── shipping/
│
├── kubernetes/
│   ├── accounting/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── add/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── cart/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── checkout/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── ...
│   └── shipping/
│       ├── deployment.yaml
│       └── service.yaml
│
└── argocd/
    └── applicationset.yaml
```

### Directory Responsibilities

**`src/`**

Contains the source code and Docker build context for the individual services.

**`kubernetes/`**

Contains the Kubernetes desired-state configuration. Each service has its own directory containing Kubernetes manifests.

**`argocd/`**

Contains the Argo CD ApplicationSet configuration used to automatically generate Argo CD Applications.

---

## CI/CD Pipeline

GitHub Actions is used to automate the Continuous Integration process for the individual services.

The pipeline performs security scanning, container image creation, vulnerability scanning, image publishing, and Kubernetes manifest updates.

### Pipeline Flow

```text
Developer
    │
    │ git push
    ▼
GitHub
    │
    ▼
GitHub Actions
    │
    ├── Gitleaks
    │      │
    │      └── Secret scanning
    │
    ├── Checkov
    │      │
    │      └── Kubernetes security scanning
    │
    ├── Docker Build
    │      │
    │      └── Build image using Git SHA
    │
    ├── Trivy
    │      │
    │      └── Container vulnerability scanning
    │
    ├── Docker Hub
    │      │
    │      └── Push image
    │
    └── Update Kubernetes Manifest
           │
           └── Commit & push image tag
                    │
                    ▼
                  GitHub
```

### Workflow Trigger

Each service pipeline is configured to run when changes affecting that service are pushed to the `main` branch.

For example, the Accounting service workflow monitors:

```yaml
on:
  push:
    branches:
      - main
    paths:
      - "src/accounting/**"
      - "kubernetes/accounting/**"
```

This prevents unrelated service changes from unnecessarily triggering the Accounting pipeline.


---

## CI Security

The pipeline integrates multiple security tools into the CI process.

### Gitleaks

**Gitleaks** scans the application source code for accidentally committed secrets such as credentials, API keys, and tokens.

```text
Source Code
     │
     ▼
 Gitleaks
     │
     ▼
Secret Detection Report
```

### Checkov

**Checkov** scans Kubernetes manifests against security and configuration best practices.

```text
Kubernetes Manifests
        │
        ▼
     Checkov
        │
        ▼
Security / Compliance Report
```

### Trivy

**Trivy** scans the newly built Docker image for known vulnerabilities, with the pipeline configured to report `HIGH` and `CRITICAL` severity findings.

```text
Docker Image
     │
     ▼
   Trivy
     │
     ▼
Vulnerability Report
```

Reports generated by the security tools are uploaded as GitHub Actions artifacts.

---

## Docker Image Build and Publishing

The Docker image is built using the Git commit SHA as its tag:

```bash
docker build \
  -t accounting-svc:${{ github.sha }} \
  -f src/accounting/Dockerfile .
```

The image is then tagged for Docker Hub:

```text
<DOCKER_USERNAME>/accounting-svc:<commit-sha>
```

and pushed to the registry:

```bash
docker push \
  ${{ secrets.DOCKER_USERNAME }}/accounting-svc:${{ github.sha }}
```

Using the Git commit SHA provides traceability between the source-code revision and the container image.

For example:

```text
Git Commit
    │
    │ abc123...
    ▼
Docker Image
    │
    └── accounting-svc:abc123...
```

---

## Kubernetes Manifest Update

After successfully building and pushing the image, GitHub Actions updates the Kubernetes Deployment manifest with the new image tag.

For example:

```yaml
image: <DOCKER_USERNAME>/accounting-svc:<commit-sha>
```

The workflow then commits the updated manifest back to the `main` branch.

This step is important because the Kubernetes manifest stored in Git represents the **desired state** of the application.

The workflow therefore follows:

```text
Build Image
     │
     ▼
Push Image to Docker Hub
     │
     ▼
Update Kubernetes Manifest
     │
     ▼
Commit & Push to GitHub
     │
     ▼
Git becomes updated desired state
```

GitHub Actions does **not** directly deploy the new version using `kubectl`.

Instead, it updates Git, and Argo CD handles the deployment.

---

# GitOps Workflow

Argo CD provides the Continuous Delivery and GitOps portion of the project.

GitHub acts as the **source of truth**, while Argo CD continuously compares the desired state stored in Git with the live state of the Kubernetes cluster.

```text
                   GitHub
                Desired State
                     │
                     │
                     ▼
                  Argo CD
                     │
              Compare / Reconcile
                     │
                     ▼
               Amazon EKS
                Live State
```

When a Kubernetes manifest changes in Git, Argo CD detects the change and synchronizes the Kubernetes cluster according to the configured synchronization policy.

### End-to-End GitOps Flow

```text
Developer
    │
    ▼
GitHub
    │
    ▼
GitHub Actions
    │
    ├── Security Scans
    ├── Docker Build
    ├── Trivy Scan
    └── Push Image
            │
            ▼
        Docker Hub
            │
            ▼
    Update K8s Manifest
            │
            ▼
         GitHub
            │
            ▼
         Argo CD
            │
            ▼
      ApplicationSet
            │
            ▼
    Argo CD Application
            │
            ▼
        Amazon EKS
```

---

# Argo CD ApplicationSet

The project uses **Argo CD ApplicationSet** to simplify the management of multiple microservices.

A **Git Directory Generator** scans the `kubernetes/` directory and discovers the individual service directories.

For example:

```text
kubernetes/
├── accounting/
├── add/
├── cart/
├── checkout/
├── payment/
├── product-catalog/
├── recommendation/
└── shipping/
```

ApplicationSet uses a common template to automatically generate an individual Argo CD Application for each discovered directory.

```text
Git Directory Generator
          │
          ▼
     ApplicationSet
          │
          ├── accounting Application
          ├── add Application
          ├── cart Application
          ├── checkout Application
          ├── payment Application
          ├── product-catalog Application
          ├── recommendation Application
          └── shipping Application
```

This eliminates the need to manually create and maintain a separate Argo CD Application manifest for every service.

It also makes the GitOps architecture easier to scale: when a new service directory is added under `kubernetes/`, the ApplicationSet can automatically discover it and generate the corresponding Argo CD Application.

---

# Prerequisites

Before deploying the project, ensure the following tools and resources are available:

* AWS account
* Amazon EKS cluster
* AWS CLI
* `kubectl`
* Helm
* Git
* Docker
* GitHub repository
* Docker Hub account
* Argo CD installed in the EKS cluster

Configure AWS credentials and verify access to the EKS cluster:

```bash
aws sts get-caller-identity
```

Configure `kubectl`:

```bash
aws eks update-kubeconfig \
  --region <AWS_REGION> \
  --name <EKS_CLUSTER_NAME>
```

Verify cluster access:

```bash
kubectl get nodes
```

---

# Deployment

## 1. Clone the Repository

```bash
git clone https://github.com/HumayunChaudhary/eks-microservices-gitops.git
cd eks-microservices-gitops
```

## 2. Configure GitHub Secrets

The GitHub Actions workflows require Docker Hub credentials.

Configure the following GitHub repository secrets:

```text
DOCKER_USERNAME
DOCKER_TOKEN
```

The workflow uses these credentials to authenticate with Docker Hub and push container images.

## 3. Install Argo CD

If Argo CD is not already installed in the EKS cluster, install it in the `argocd` namespace.

```bash
kubectl create namespace argocd
```

Install Argo CD using Helm:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  -n argocd
```

Verify the installation:

```bash
kubectl get pods -n argocd
```

## 4. Apply the ApplicationSet

Once Argo CD is running, apply the ApplicationSet:

```bash
kubectl apply -f argocd/applicationset.yaml
```

Verify the ApplicationSet:

```bash
kubectl get applicationset -n argocd
```

Verify generated Applications:

```bash
kubectl get applications -n argocd
```

The ApplicationSet should automatically generate Applications based on the directories under:

```text
kubernetes/
```

---

# Verification

Verify the generated Argo CD Applications:

```bash
kubectl get applications -n argocd
```

Verify the Kubernetes workloads:

```bash
kubectl get deployments -A
```

Verify the running pods:

```bash
kubectl get pods -A
```

Verify Kubernetes Services:

```bash
kubectl get services -A
```

For a specific service:

```bash
kubectl get pods -n default
kubectl get deployment <deployment-name> -n default
kubectl get service <service-name> -n default
```

---

# GitOps Benefits

This implementation provides several benefits over manually deploying Kubernetes manifests:

### Single Source of Truth

Git stores the desired Kubernetes configuration, providing a version-controlled and auditable deployment history.

### Automated Deployment

Argo CD can automatically synchronize changes from Git into the Kubernetes cluster.

### Self-Healing

Argo CD can detect configuration drift and restore resources to the desired state defined in Git.

### Automated Pruning

Resources removed from the desired state can be automatically removed from the cluster when pruning is enabled.

### Scalable Application Management

ApplicationSet eliminates repetitive Argo CD Application manifests by dynamically generating Applications from service directories.

### Traceable Container Versions

Docker images are tagged with Git commit SHAs, making it possible to trace a running image back to the source-code revision that produced it.

### Security Integrated into CI

Gitleaks, Checkov, and Trivy integrate security checks directly into the CI pipeline.

---

# CI/CD vs GitOps Responsibilities

The architecture deliberately separates CI responsibilities from CD responsibilities.

| Component      | Responsibility                           |
| -------------- | ---------------------------------------- |
| GitHub         | Source code and Kubernetes desired state |
| GitHub Actions | CI automation and security scanning      |
| Gitleaks       | Secret detection                         |
| Checkov        | Kubernetes security scanning             |
| Docker         | Container image creation                 |
| Trivy          | Container vulnerability scanning         |
| Docker Hub     | Container image registry                 |
| Argo CD        | GitOps-based continuous delivery         |
| ApplicationSet | Dynamic Argo CD Application generation   |
| Amazon EKS     | Kubernetes workload execution            |

The overall responsibility model is:

```text
                  CI
                   │
                   ▼
             GitHub Actions
                   │
          Build + Scan + Push
                   │
                   ▼
              Docker Hub
                   │
                   │
                   │
             GitOps / CD
                   │
                   ▼
                GitHub
                   │
                   ▼
                Argo CD
                   │
                   ▼
              Amazon EKS
```

---

# Future Improvements

Potential improvements to the project include:

* Introduce separate `dev`, `staging`, and `production` environments.
* Use Kustomize or Helm for environment-specific configuration.
* Implement progressive delivery using Argo Rollouts.
* Introduce blue-green or canary deployment strategies.
* Add image signing and verification.
* Implement stronger policy enforcement using Kubernetes admission policies.
* Integrate automated rollback strategies.
* Introduce centralized secrets management using AWS Secrets Manager and External Secrets.
* Add automated testing stages before container image publishing.

---

# Conclusion

This project demonstrates a complete modern DevOps workflow for deploying a microservices application on Amazon EKS.

The implementation combines:

```text
GitHub
   +
GitHub Actions
   +
Docker
   +
Docker Hub
   +
Kubernetes
   +
Amazon EKS
   +
Argo CD
   +
ApplicationSet
```

The resulting architecture provides an automated and scalable deployment model where **GitHub defines the desired state, GitHub Actions builds and secures application artifacts, Docker Hub stores container images, and Argo CD continuously reconciles Kubernetes workloads running on Amazon EKS with the state defined in Git.**
