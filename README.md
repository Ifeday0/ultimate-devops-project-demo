# OpenTelemetry Astronomy Shop --- AWS EKS Deployment

A hands-on DevOps project demonstrating an end-to-end CI/CD and GitOps
deployment of selected services from the **OpenTelemetry Astronomy
Shop**---a microservice-based distributed application designed to
demonstrate OpenTelemetry in a near-real-world environment.

The project focuses on taking selected application services from source
code to a running Kubernetes workload on **Amazon EKS**, with automated
build, container publishing, manifest updates, and deployment
synchronization.

> **Project scope:** This implementation uses approximately three
> selected microservices to demonstrate the complete delivery workflow,
> rather than deploying and customizing every service in the Astronomy
> Shop.

🌐 **Live application:** <https://ifecloudevopsmaster.online>\
💻 **My repository:**
[Ifeday0/ultimate-devops-project-demo](https://github.com/Ifeday0/ultimate-devops-project-demo)\
📦 **Original project:**
[iam-veeramalla/ultimate-devops-project-demo](https://github.com/iam-veeramalla/ultimate-devops-project-demo)\
🎥 **Video walkthrough:** *Add your video link here*

------------------------------------------------------------------------

## ✨ Project Highlights

-   ☁️ Provisioned AWS networking and an Amazon EKS cluster using
    Terraform.
-   🐳 Containerized selected application services with Docker.
-   ⚙️ Implemented a GitHub Actions CI/CD workflow for build, test,
    code-quality checks, and image publishing.
-   🚀 Used Argo CD to continuously reconcile Kubernetes manifests from
    Git.
-   🗂️ Stored and published container images to Docker Hub.
-   🌍 Connected a custom domain using GoDaddy DNS delegation and Amazon
    Route 53.
-   🔐 Configured HTTPS with AWS Certificate Manager (ACM) and an
    Application Load Balancer (ALB).
-   📈 Used the OpenTelemetry Astronomy Shop as a distributed,
    microservice-based application example.

## 🧩 Application Overview

The Astronomy Shop is a distributed application made up of independently
deployable services. This project focuses on a small selection of
services to demonstrate how application code moves through a modern
delivery pipeline and runs on Kubernetes.

The diagram below is a simplified view of the application path relevant
to this deployment. It is not intended to represent every service in the
upstream Astronomy Shop.

``` mermaid
flowchart TD
    U[User / Browser] --> DNS[Custom Domain]
    DNS --> ALB[Application Load Balancer]
    ALB --> FP[Frontend Proxy]
    FP --> FE[Frontend Application]
    FE --> PC[Product Catalog Service]
    FE --> S2[Selected Microservice 2]
    FE --> S3[Selected Microservice 3]
```

## 🏗️ AWS and Deployment Architecture

The infrastructure and delivery workflow are separated into two parts:
Terraform provisions the AWS platform, while GitHub Actions and Argo CD
automate application delivery.

``` mermaid
flowchart TB
    subgraph Source["Source Control"]
        GH[GitHub Repository]
    end

    subgraph Pipeline["GitHub Actions CI/CD"]
        CI[Build and Unit Tests]
        CQ[Code Quality Checks]
        IMG[Build Docker Image]
        DH[Push Image to Docker Hub]
        MAN[Update Kubernetes Manifest]
        CI --> IMG --> DH --> MAN
        CQ -. Independent quality check .-> CI
    end

    subgraph AWS["AWS Cloud"]
        R53[Route 53 Hosted Zone]
        ACM[ACM TLS Certificate]
        ALB[Application Load Balancer]
        subgraph VPC["Terraform-provisioned VPC"]
            EKS[Amazon EKS Cluster]
            NODES[Worker Nodes]
            PODS[Selected Application Pods]
            EKS --> NODES --> PODS
        end
        R53 --> ALB
        ACM --> ALB
        ALB --> PODS
    end

    subgraph GitOps["GitOps Deployment"]
        ARGO[Argo CD]
    end

    GH --> CI
    MAN --> GH
    GH --> ARGO
    ARGO --> EKS
    DH -. Kubernetes pulls image at runtime .-> PODS
```

### 🧱 Infrastructure provisioning

Terraform was used to provision the foundational AWS infrastructure,
including:

-   **VPC and networking:** Created the network environment and subnets
    required by the EKS deployment.
-   **Amazon EKS:** Provisioned the Kubernetes control plane and worker
    capacity.
-   **Kubernetes deployment platform:** Prepared the cluster for
    application workloads and the AWS Load Balancer Controller.

The infrastructure-as-code approach makes the environment easier to
reproduce and manage than manually creating each resource in the AWS
console.

### 🌐 Domain, DNS and HTTPS

The application was exposed through a custom domain:

1.  Registered the domain with GoDaddy.
2.  Created a public hosted zone in Amazon Route 53.
3.  Updated the domain's nameserver records at GoDaddy to delegate DNS
    management to Route 53.
4.  Created Route 53 Alias records for the root domain and `www`,
    pointing to the application load balancer.
5.  Requested and DNS-validated an SSL/TLS certificate in AWS
    Certificate Manager.
6.  Configured the Kubernetes Ingress with the ALB certificate, HTTP and
    HTTPS listeners, and an HTTP-to-HTTPS redirect.

**Result:** Both `https://ifecloudevopsmaster.online` and
`https://www.ifecloudevopsmaster.online` returned successful HTTPS
responses.

TLS terminates at the ALB. The ALB forwards traffic to the frontend
proxy service in the EKS cluster.

## 🔄 CI/CD and GitOps Workflow

The delivery process demonstrates the separation between continuous
integration, image publishing, and Kubernetes deployment.

### 1. Continuous Integration --- GitHub Actions

A push or pull request targeting `main` triggers the workflow:

-   Checks out the repository.
-   Sets up the Go environment.
-   Downloads dependencies.
-   Builds the Product Catalog service.
-   Runs Go unit tests.
-   Runs code-quality checks in a separate job.

The code-quality job is independent of the deployment jobs. A lint
failure is reported, but it does not prevent the Docker publishing and
manifest-update jobs from running when the build and tests succeed.

### 2. Containerization and image publishing


After a successful build on `main`:

-   Docker Buildx builds the service image.
-   GitHub Actions authenticates to Docker Hub using repository secrets.
-   The image is pushed to Docker Hub with a unique GitHub Actions run
    ID as its tag.

Example image tag format:

``` text
<docker-hub-username>/product-catalog:<github-run-id>
```

### 3. Continuous Deployment --- Argo CD

After the image is published:

1.  GitHub Actions updates the Product Catalog Kubernetes deployment
    manifest with the new image tag.
2.  The workflow commits and pushes the manifest change to the GitHub
    repository.
3.  Argo CD detects the Git change and synchronizes the desired
    Kubernetes configuration with the EKS cluster.
4.  Kubernetes schedules the updated workload and the node's container
    runtime pulls the image from Docker Hub.

**Important:** Argo CD synchronizes Kubernetes configuration from Git.
It does not pull the container image itself; Kubernetes pulls the image
when creating the Pod.

``` mermaid
sequenceDiagram
    participant Dev as Developer
    participant GH as GitHub
    participant GA as GitHub Actions
    participant DH as Docker Hub
    participant Argo as Argo CD
    participant EKS as Kubernetes on EKS

    Dev->>GH: Push code to main
    GH->>GA: Trigger workflow
    GA->>GA: Build and test
    GA->>DH: Build and push tagged image
    GA->>GH: Commit updated deployment manifest
    Argo->>GH: Detect manifest change
    Argo->>EKS: Sync desired configuration
    EKS->>DH: Pull container image
    EKS->>EKS: Start updated Pod
```

## 🧰 Technology Stack

  -----------------------------------------------------------------------
  Area                                Tools / Services
  ----------------------------------- -----------------------------------
  Application                         OpenTelemetry Astronomy Shop, Go
                                      service

  Source control                      Git, GitHub

  Infrastructure as Code              Terraform

  Cloud platform                      AWS

  Containerization                    Docker, Docker Buildx

  Container registry                  Docker Hub

  CI                                  GitHub Actions

  CD / GitOps                         Argo CD

  Orchestration                       Kubernetes, Amazon EKS

  Ingress and load balancing          Kubernetes Ingress, AWS Load
                                      Balancer Controller, ALB

  DNS                                 GoDaddy, Amazon Route 53

  TLS                                 AWS Certificate Manager
  -----------------------------------------------------------------------

## 🧠 Challenges and Solutions

  -----------------------------------------------------------------------
  Challenge                           How it was addressed
  ----------------------------------- -----------------------------------
  Accessing the application through   Investigated host-based Ingress
  the ALB initially returned a 404    routing and tested requests using
  response.                           the expected host header.
                                      Configured the custom domain to
                                      route to the ALB.

  Custom domain was not resolving     Created a Route 53 hosted zone and
  through AWS.                        updated the domain nameservers at
                                      GoDaddy to delegate DNS to Route
                                      53.

  Enabling HTTPS for the custom       Requested an ACM certificate,
  domain.                             completed DNS validation, and
                                      attached the certificate to the ALB
                                      HTTPS listener through Ingress
                                      annotations.

  GitHub Actions workflow did not     Corrected the pull request
  initially run for the intended pull base/compare branch selection so
  request.                            the feature branch targeted `main`.

  Docker Hub authentication failed in Regenerated a Docker Hub access
  the pipeline.                       token with the required permissions
                                      and updated the GitHub repository
                                      secrets.

  Code-quality failures could block   Separated code-quality checks from
  the deployment pipeline.            the deployment dependency chain so
                                      image publishing can proceed when
                                      build and tests pass.

  Argo CD reported the Product        Investigated ReplicaSets and
  Catalog application as Degraded     scheduler events. The worker nodes
  because a new Pod could not be      had a Pod capacity of 17 each and
  scheduled.                          were reporting `Too many pods`; the
                                      image tag itself was correct. This
                                      identified a cluster
                                      scheduling-capacity constraint
                                      rather than an image-version
                                      problem.
  -----------------------------------------------------------------------

## ✅ Verification

The following checks were used during implementation:

``` bash
# Check cluster nodes
kubectl get nodes -o wide

# Check application resources
kubectl get applications -n argocd

# Check Pods across namespaces and their assigned nodes
kubectl get pods -A -o wide

# Check the application Ingress
kubectl get ingress

# Verify HTTP redirects to HTTPS
curl -I http://ifecloudevopsmaster.online

# Verify HTTPS response
curl -I https://ifecloudevopsmaster.online
curl -I https://www.ifecloudevopsmaster.online
```

The custom domain returned successful HTTPS responses. The Product
Catalog rollout also exposed a worker-node Pod-capacity limit during
scheduling; this is documented as an infrastructure constraint
discovered during validation.

## 🎥 Video Demonstration

*Add your video link or embed location here.*

The walkthrough will demonstrate the deployed application, AWS
infrastructure, GitHub Actions pipeline, Docker Hub image, Kubernetes
manifests, and Argo CD synchronization.

## 📌 Project Outcome

This project demonstrates a practical end-to-end DevOps workflow:
provisioning AWS infrastructure with Terraform, packaging application
services into Docker images, automating build and test workflows with
GitHub Actions, publishing images to Docker Hub, and deploying
Kubernetes configuration through Argo CD on Amazon EKS.

It also provided hands-on experience troubleshooting real deployment
concerns involving DNS delegation, TLS certificates, ALB/Ingress
routing, CI workflow triggers, registry authentication, GitOps
synchronization, and Kubernetes scheduling capacity.

------------------------------------------------------------------------

**Note:** This is a focused implementation of selected services from the
upstream OpenTelemetry Astronomy Shop project, not a full deployment of
every microservice in the original application.
