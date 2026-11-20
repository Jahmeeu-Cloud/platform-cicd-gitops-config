# Platform CI/CD GitOps Configuration with Kubernetes, Kustomize and ArgoCD

---

## Project Overview

This repository contains the GitOps configuration for a Kubernetes application deployment managed using **Kubernetes, Kustomize and ArgoCD**.

The objective of this project was to understand how Kubernetes deployment configurations can be maintained declaratively in Git, monitored by ArgoCD, and automatically reconciled with the actual state of a Kubernetes cluster.

Rather than relying entirely on manual deployment commands, I created a separate GitOps repository to manage the application's desired deployment state.

The repository separates application source code from Kubernetes deployment configuration, making deployment changes easier to track, review, maintain and reproduce.

One of the most important parts of this project was testing ArgoCD self-healing by deliberately changing the replica count in the running Kubernetes Deployment and observing ArgoCD restore the configuration defined in Git.

This provided practical experience with declarative infrastructure management, configuration drift detection and automated reconciliation.

## 1. Project Objectives

The main objectives were:

- Maintain Kubernetes deployment manifests in a dedicated Git repository.
- Separate application source code from deployment configuration.
- Define application workloads using Kubernetes YAML manifests.
- Organise deployment resources using Kustomize.
- Connect the GitOps repository to ArgoCD.
- Monitor application synchronisation and health.
- Enable automated synchronisation and self-healing.
- Test configuration drift and automatic reconciliation.
- Build a reusable foundation for future multi-environment deployments.

## 2. Architecture

The project follows a GitOps workflow in which Git serves as the source of truth for the desired Kubernetes deployment configuration.

### Architecture Flow

```text
       Application Source Code
                 |
                 v
          Jenkins CI Pipeline
                 |
                 v
          Docker Image Build
                 |
                 v
             Docker Hub
                 |
                 v
       GitOps Configuration Repo
                 |
                 v
              ArgoCD
                 |
          Git Repository
          Monitoring
                 |
                 v
        Desired Kubernetes State
                 |
                 v
          Kubernetes Cluster
                 |
                 v
          Spring Boot App
```

**Important architecture distinction:**

Jenkins runs on an AWS EC2 instance, while Kubernetes and ArgoCD run in a local Minikube environment.

The GitOps approach allows ArgoCD to manage cluster-side deployments without requiring the remote Jenkins server to connect directly to the developer's local Kubernetes API.

The image publication and GitOps configuration update are separate operations. A new Docker image must be referenced by a corresponding Git configuration change or an image-update automation mechanism before ArgoCD can deploy it.

## 3. Repository Structure

The confirmed repository structure contains the Spring Boot application's Kubernetes base configuration.

```text
platform-cicd-gitops-config/
│
└── apps/
    └── spring-boot-app/
        └── base/
            ├── deployment.yaml
            ├── service.yaml
            └── kustomization.yaml
```

### Purpose of Each File

| File | Responsibility |
|---|---|
| deployment.yaml | Defines the Spring Boot application Deployment and desired replica count |
| service.yaml | Exposes the application internally or through the configured NodePort |
| kustomization.yaml | Identifies and organises Kubernetes resources for Kustomize |

This separation provides a clean foundation for managing application deployment configurations independently of application development.

## 4. Kubernetes Deployment Configuration

The Kubernetes Deployment defines the desired state of the Spring Boot application.

The confirmed configuration includes:

- Application: spring-boot-app
- Container image: horlacloud/spring-boot-demo:20
- Desired replicas: 3
- Container port: 8080

The Deployment allows Kubernetes to maintain the requested number of application Pods.

When a Pod fails or is terminated, Kubernetes attempts to create a replacement to maintain the desired replica count.

### Kubernetes Service

The application is exposed through a Kubernetes Service named:

`spring-boot-app-service`

The Service provides a stable endpoint for accessing the application Pods.

Its configuration maps:

- Service port: 80
- Target container port: 8080
- Service type: NodePort

This allows the application to be accessed through the configured Minikube networking mechanism.

## 5. Kustomize Configuration

Kustomize was used to organise and render the Kubernetes deployment configuration.

The base directory contains the resources required by the Spring Boot application.

To render the configuration, run:

```bash
kubectl kustomize apps/spring-boot-app/base
```

This generates the combined Kubernetes configuration without requiring manual concatenation of the individual YAML files.

Before committing changes, I use Git to review the configuration:

```bash
git status
git diff
```

This helps identify unintended modifications before pushing deployment changes to the remote repository.

Kustomize also provides a foundation for future environment-specific configurations, such as development, staging and production overlays.

## 6. ArgoCD GitOps Integration

ArgoCD is responsible for monitoring the GitOps repository and reconciling the Kubernetes cluster with the desired configuration stored in Git.

The application configuration uses the following settings:

| Configuration | Value |
|---|---|
| Application | spring-boot-app |
| Repository | platform-cicd-gitops-config |
| Branch | main |
| Manifest path | apps/spring-boot-app/base |
| Destination | Kubernetes cluster |
| Namespace | default |
| Deployment replicas | 3 |

ArgoCD continuously compares the desired configuration in Git with the live Kubernetes resources it manages.

When a configuration change is committed and pushed, ArgoCD can detect the difference and synchronise the cluster.

### Inspect ArgoCD Application

```bash
kubectl get applications -n argocd
```

To inspect the specific application:

```bash
kubectl get application spring-boot-app -n argocd
```

The expected successful application status is:

- **Synced:** The managed resources match the desired configuration.
- **Healthy:** The application resources pass ArgoCD's health assessment.

These statuses provide useful deployment visibility, although application-level testing is still necessary.

## 7. Testing ArgoCD Self-Healing and Configuration Drift

One of the most valuable exercises in this project was testing ArgoCD's automated self-healing capability.

The GitOps repository declared that the application should run with three replicas.

I deliberately changed the live Kubernetes Deployment to one replica:

```bash
kubectl scale deployment spring-boot-app --replicas=1
```

I then monitored the Deployment:

```bash
kubectl get deployment spring-boot-app --watch
```

Because ArgoCD was configured for automated synchronisation and self-healing, it detected that the live replica count no longer matched the desired state defined in Git.

ArgoCD reconciled the Deployment, restoring the desired replica count to three.

### What This Demonstrated

1. Git maintained the desired application configuration.
2. A manual change introduced configuration drift.
3. ArgoCD detected the difference between Git and the live cluster.
4. Automated reconciliation restored the Git-defined configuration.
5. Kubernetes worked towards maintaining the desired three replicas.

This exercise demonstrated a practical GitOps principle: the desired state is maintained in version control, while the GitOps controller continuously works to reconcile the running environment with that state.

## 8. Kubernetes Deployment Validation

After synchronisation, I used Kubernetes commands to inspect the application resources.

### Check Deployment

```bash
kubectl get deployment spring-boot-app
```

### Check Pods

```bash
kubectl get pods -o wide
```

### Check Service

```bash
kubectl get service spring-boot-app-service
```

### Check Service Endpoints

```bash
kubectl get endpoints spring-boot-app-service
```

### Inspect ArgoCD

```bash
kubectl get application spring-boot-app -n argocd
```

These checks help verify that the expected Kubernetes resources exist, the application Pods become ready, the Service has suitable endpoints and ArgoCD reports the expected application status.

The deployed application was also tested through its Minikube access endpoint, returning **HTTP 200**.

## 9. GitOps Repository Management

The repository is maintained using Git to provide version history and traceability for deployment configuration changes.

One of the confirmed commits from the project was:

```text
dc9e323 Fix deployment YAML and set replicas to three
```

Useful Git commands for maintaining the repository include:

```bash
git status

git log -5 --oneline

git diff

git add apps/spring-boot-app/base

git commit -m "Update application deployment configuration"

git push origin main
```

Keeping deployment configuration in Git makes it easier to review changes, identify previous configurations and understand how the desired application state evolved.

## 10. Security and Operational Considerations

For a more production-oriented GitOps implementation, I would prioritise:

- Protecting the GitOps repository and deployment branch.
- Using restricted credentials for private repository access.
- Keeping application secrets outside ordinary Git manifests.
- Reviewing automated pruning before enabling it.
- Applying Kubernetes RBAC and least-privilege access.
- Using immutable container image tags or image digests.
- Validating Kubernetes manifests before deployment.
- Implementing appropriate health checks and resource limits.
- Introducing monitoring, logging and alerting.
- Establishing tested rollback and recovery procedures.

These are recommended improvements and should be implemented and validated according to the target environment.

## 11. Future Improvements

The current repository provides a foundation for expanding the GitOps workflow.

Potential improvements include:

1. **Multi-environment deployment:** Introduce development, staging and production overlays using Kustomize.
2. **Automated image updates:** Configure Jenkins to update the GitOps repository after successfully publishing a new image.
3. **Pull-request validation:** Validate Kubernetes manifests before merging configuration changes.
4. **Application security:** Integrate container image scanning and dependency security checks.
5. **Observability:** Integrate Prometheus and Grafana for monitoring Kubernetes workloads.
6. **Progressive delivery:** Explore canary and blue-green deployment strategies.
7. **Secrets management:** Integrate an appropriate external secrets-management solution.
8. **Production Kubernetes:** Extend the workflow to a multi-node cluster with appropriate availability and security controls.

These improvements would help evolve the repository into a more comprehensive platform deployment configuration.

## 12. What I Learned

Working on this project strengthened my practical understanding of:

- Declarative Kubernetes configuration.
- Git as the source of truth for deployment state.
- Kustomize resource organisation.
- ArgoCD application management.
- Automated synchronisation and reconciliation.
- Configuration drift detection.
- Kubernetes Deployment and Service management.
- Application deployment validation.
- Git-based configuration history and change management.

The most important lesson was understanding that GitOps is not simply about storing Kubernetes YAML files in Git. It is about establishing a reliable relationship between the desired state declared in version control and the actual state running inside a Kubernetes environment.

## Conclusion

This project demonstrates how Kubernetes deployment configuration can be managed through a dedicated GitOps repository using Kustomize and ArgoCD.

By separating application source code from deployment configuration, I established a clearer deployment management workflow and gained practical experience with automated reconciliation and self-healing.

The self-healing test was particularly valuable because it demonstrated how ArgoCD can detect configuration drift and restore the desired state without relying on manual intervention.

This repository represents an important foundation in my continued development as a Cloud, DevOps and Platform Engineer.

---

## Project Resources

**GitHub Repository – GitOps Configuration**

https://github.com/Jahmeeu-Cloud/platform-cicd-gitops-config

**GitHub Repository – Application Source Code**

https://github.com/Jahmeeu-Cloud/platform-cicd-spring-boot-demo-source

**GitHub Profile**

https://github.com/Jahmeeu-Cloud

**Full Project Documentation – Dev.to**

(https://dev.to/jamiu_cloud/building-a-cicd-and-gitops-platform-with-jenkins-docker-kubernetes-and-argo-cd-my-end-to-end-1nkg)

**Portfolio**

https://jahmeeu-cloud.github.io/portfolio-project/

**Docker Hub**

https://hub.docker.com/u/horlacloud

---
#devops #platformengineering #kubernetes #gitops #argocd #kustomize #cicd

**Author:** Jamiu Olatunji Bakre  
**Role:** Cloud, DevOps & Platform Engineer  
**Project:** Platform CI/CD GitOps Configuration
*Building practical cloud infrastructure, automated delivery pipelines, Kubernetes platforms and GitOps workflows.*
