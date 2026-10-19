# Microservices on Azure Kubernetes Service: CI/CD Pipeline

An e-commerce application of Python FastAPI microservices, deployed end to end on **Microsoft Azure**: infrastructure as code, container images in Azure Container Registry, **Kubernetes on AKS**, **GitHub Actions** CI/CD, and Prometheus + Grafana monitoring.

Completed in **SIT722 Software Deployment and Operation** at Deakin University (July – October 2025).

> **Credit:** The application code (FastAPI services, frontend and base manifests) is starter code from the SIT722 teaching team. My work was provisioning, deploying and operating it: configuring the infrastructure, fixing and running the pipelines, and deploying to Azure. See [My work](#my-work).

**Stack:** Azure (AKS, ACR, Storage) · OpenTofu/Terraform · Docker · Kubernetes · GitHub Actions · Python FastAPI · PostgreSQL · Nginx · Prometheus · Grafana

## Architecture

```
 GitHub push ──► CI (GitHub Actions)                 CD (GitHub Actions, manual trigger)
                 ├─ start PostgreSQL test services   ├─ az login (service principal)
                 ├─ pytest for each service          ├─ get AKS credentials, attach ACR
                 ├─ docker build                     ├─ create K8s secret from GitHub Secrets
                 └─ push images to ACR ─────────────►├─ kubectl apply: DBs, services, frontend
                                                     └─ wait for LoadBalancer IPs, inject into frontend

 Azure (provisioned with OpenTofu in infra/)
 ┌──────────────────────── Resource group ─────────────────────────┐
 │  ACR (images)   Storage account (product images)                │
 │  AKS cluster:  frontend (Nginx) ─► product-service ─► product-db │
 │                                 └► order-service   ─► order-db   │
 │                Prometheus + Grafana (monitoring/)               │
 └─────────────────────────────────────────────────────────────────┘
```

## Repository layout

| Path | Contents |
| --- | --- |
| `infra/` | OpenTofu (Terraform) for the resource group, ACR, AKS cluster and storage account |
| `backend/product_service`, `backend/order_service` | FastAPI microservices with pytest suites and Dockerfiles |
| `frontend/` | Static frontend served by Nginx |
| `k8s/` | Kubernetes Deployments, Services, ConfigMaps and the secret template |
| `.github/workflows/` | CI (test, build, push to ACR) and CD (deploy to AKS) for backend and frontend |
| `monitoring/` | Prometheus (with RBAC and pod discovery) and Grafana manifests |
| `docker-compose.yml` | Run the whole stack locally |

## My work
- **Provisioned Azure with OpenTofu:** resource group, container registry, AKS cluster and storage account, using my own resource prefix (`infra/variables.tf`).
- **Fixed the CI/CD pipelines:** corrected ACR authentication and image naming across all four workflows, deriving the login server (`<registry>.azurecr.io`) from a single registry-name secret.
- **Deployed to AKS through GitHub Actions:** service-principal login, ACR attached to the cluster, rollout of the databases and services, then the frontend wired to the backend's LoadBalancer IPs.
- **Configured the Kubernetes manifests** to pull images from my registry.
- **Ran the stack in earlier labs:** locally with Docker/Podman and Docker Desktop Kubernetes, and pushed images to Docker Hub.
- **Monitoring:** deployed Prometheus (Kubernetes pod discovery) and Grafana to the cluster.
- **DevSecOps:** scanned the container images with Docker Scout and wrote up remediation (base-image hardening, dependency updates, secret management).

### Portfolio clean-up
The original coursework stored the Kubernetes secret, including an Azure Storage key, in `k8s/secrets.yaml`. In this copy, the CD workflow builds the secret at deploy time from **GitHub Secrets**, and only a template (`k8s/secrets.example.yaml`) is committed.

## Run it

### Locally
```bash
docker compose up --build
# frontend at http://localhost:3000; services and ports are listed in docker-compose.yml
```

### On Azure
1. Provision infrastructure:
   ```bash
   cd infra
   tofu init && tofu plan -out tfplan && tofu apply tfplan
   ```
2. In the GitHub repository, add these Actions secrets: `AZURE_CREDENTIALS` (service principal JSON), `AZURE_CONTAINER_REGISTRY` (registry name), `POSTGRES_USER`, `POSTGRES_PASSWORD`, `AZURE_STORAGE_ACCOUNT_NAME`, `AZURE_STORAGE_ACCOUNT_KEY`.
3. Push to `main` to run CI (tests, build and push to ACR).
4. Run **CD - Deploy Backend Services to AKS** (manual trigger), then the frontend CD workflow.
5. Monitoring: `kubectl apply -f monitoring/`. Grafana uses default credentials here, so change them before any real use.

## What I'd improve
- Tag images with the commit SHA (the workflows already define `IMAGE_TAG`) instead of `latest`, so each deployment can be traced and rolled back
- Use Azure Key Vault (or Workload Identity) instead of storage account keys
- Gate CD on CI success instead of a manual trigger

## What I learned
- Debugging failed GitHub Actions runs from their logs, especially ACR login and image-tag mismatches
- How Deployments, Services, ConfigMaps and Secrets fit together, and why secrets should never be committed
- Reproducible infrastructure with OpenTofu, and tearing it down to control cloud costs
