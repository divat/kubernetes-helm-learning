# Kubernetes Deployment Using Helm

## Project Overview

This project demonstrates how to deploy and manage multiple Spring Boot microservices on Kubernetes using **Helm Charts**.

The solution uses a single reusable Helm chart to deploy a Product Catalog API and an Order Service across separate DEV and QA environments.

The project focuses on environment-specific configuration, reusable Kubernetes templates, Helm release management, PostgreSQL integration, Ingress-based traffic routing, and deployment version control.

## Architecture

```text
                  Client / Browser
                         |
                         v
                F5 NGINX Ingress
                  Controller
                         |
                 Kubernetes Ingress
                         |
              +----------+----------+
              |                     |
        /api/products           /api/orders
              |                     |
              v                     v
       Product API            Order Service
          :8084                   :8085
              |                     |
              +----------+----------+
                         |
                         v
                 PostgreSQL Service
                         |
                         v
                    PostgreSQL
```

### Environment Architecture

The same Helm chart is used for both environments, with separate namespaces and configuration overrides.

| Component | DEV | QA |
|---|---|---|
| Helm Release | `product-order-dev` | `product-order-qa` |
| Namespace | `helm-dev` | `helm-qa` |
| Product API | 2 replicas | 1 replica |
| Order Service | 1 replica | 2 replicas |
| Product Database | `productdb_dev` | `productdb_qa` |
| Order Database | `orderdb_dev` | `orderdb_qa` |
| Ingress Host | `dev.localenv` | `qa.localenv` |

## Technology Stack

- Java and Spring Boot
- Docker
- Kubernetes
- Helm 3
- Kind Kubernetes Cluster
- F5 NGINX Ingress Controller
- PostgreSQL
- Git and GitHub
- Ubuntu Linux

## Project Structure

```text
product-order-helm/
│
├── README.md
│
└── product-order/
    │
    ├── Chart.yaml
    ├── values.yaml
    ├── values-dev.yaml
    ├── values-qa.yaml
    │
    └── templates/
        ├── _helpers.tpl
        ├── product-api-deployment.yaml
        ├── product-api-service.yaml
        ├── order-service-deployment.yaml
        ├── order-service-service.yaml
        └── ingress.yaml
```

### Helm Chart Files

| File | Responsibility |
|---|---|
| `Chart.yaml` | Chart metadata and version |
| `values.yaml` | Common default configuration |
| `values-dev.yaml` | DEV-specific configuration |
| `values-qa.yaml` | QA-specific configuration |
| `_helpers.tpl` | Reusable Helm template helpers |
| `product-api-deployment.yaml` | Product API Deployment |
| `product-api-service.yaml` | Product API ClusterIP Service |
| `order-service-deployment.yaml` | Order Service Deployment |
| `order-service-service.yaml` | Order Service ClusterIP Service |
| `ingress.yaml` | Environment-specific Ingress routing |

## Key Helm Concepts Implemented

### 1. Reusable Helm Chart

Created a single Helm chart containing Kubernetes Deployment, Service and Ingress templates.

Used Helm template functions and built-in objects such as:

- `.Values`
- `.Release.Name`
- `.Release.Namespace`
- `.Chart.Version`
- `include`
- `define`
- `nindent`

This avoids maintaining separate Kubernetes manifests for each environment.

### 2. Environment-Specific Configuration

Common configurations are maintained in `values.yaml`.

Environment-specific differences are maintained separately:

- `values-dev.yaml`
- `values-qa.yaml`

Helm merges these override files with the common defaults during installation and upgrade.

### 3. PostgreSQL Integration

Both Spring Boot services connect to PostgreSQL using Kubernetes Service DNS.

Database configuration is supplied through Helm values.

Database credentials are referenced through Kubernetes Secrets rather than hardcoded in the Helm templates.

Separate databases are configured for DEV and QA.

### 4. Ingress and External Traffic Routing

Integrated the existing F5 NGINX Ingress Controller with Kubernetes Ingress resources.

Implemented host-based and path-based routing.

| Environment | Hostname | Path | Backend |
|---|---|---|---|
| DEV | `dev.localenv` | `/api/products` | Product API |
| DEV | `dev.localenv` | `/api/orders` | Order Service |
| QA | `qa.localenv` | `/api/products` | Product API |
| QA | `qa.localenv` | `/api/orders` | Order Service |

The Ingress controller routes incoming requests to the appropriate Kubernetes Services.

### 5. Helm Release Management

Implemented Helm release lifecycle operations:

- Install releases
- Upgrade existing releases
- Inspect release status
- View release history
- Roll back to previous revisions
- Manage separate releases across namespaces

### 6. Git Version Control

The Helm chart and environment-specific configuration files are maintained in Git.

This supports configuration history, code reviews, change tracking and future CI/CD integration.

## Prerequisites

- Ubuntu Linux environment
- Docker
- Kind
- kubectl
- Helm 3
- Existing PostgreSQL deployment
- Existing F5 NGINX Ingress Controller
- Docker images for Product API and Order Service

## Deployment Instructions

### 1. Navigate to the Helm Chart

```bash
cd product-order
```

### 2. Validate the Chart

```bash
helm lint .
```

### 3. Preview DEV Kubernetes Manifests

```bash
helm template product-order-dev . \
  -f values-dev.yaml \
  --namespace helm-dev
```

### 4. Deploy DEV

```bash
helm upgrade --install product-order-dev . \
  -f values-dev.yaml \
  --namespace helm-dev \
  --create-namespace
```

### 5. Deploy QA

```bash
helm upgrade --install product-order-qa . \
  -f values-qa.yaml \
  --namespace helm-qa \
  --create-namespace
```

**Note:** PostgreSQL credentials must be available as Kubernetes Secrets in the respective application namespaces before the application Pods start. The existing PostgreSQL deployment and databases are managed separately from this Helm chart.

## Helm Release Management Commands

List all releases:

```bash
helm list -A
```

Check release status:

```bash
helm status product-order-dev -n helm-dev
```

View release history:

```bash
helm history product-order-dev -n helm-dev
```

Upgrade DEV:

```bash
helm upgrade product-order-dev . \
  -f values-dev.yaml \
  -n helm-dev
```

Rollback DEV to revision 1:

```bash
helm rollback product-order-dev 1 -n helm-dev
```

## Kubernetes Verification

Check application Pods:

```bash
kubectl get pods -n helm-dev
kubectl get pods -n helm-qa
```

Check Services:

```bash
kubectl get svc -n helm-dev
kubectl get svc -n helm-qa
```

Check Ingress:

```bash
kubectl get ingress -n helm-dev
kubectl get ingress -n helm-qa
```

Check Service endpoints:

```bash
kubectl get endpoints -n helm-dev
kubectl get endpoints -n helm-qa
```

## Ingress Testing

For local testing, the existing F5 NGINX Ingress Controller can be accessed using port-forwarding:

```bash
kubectl port-forward -n nginx-ingress \
  svc/nginx-ingress 8080:80
```

Test DEV Product API:

```bash
curl -i -H "Host: dev.localenv" \
http://localhost:8080/api/products/health
```

Test QA Product API:

```bash
curl -i -H "Host: qa.localenv" \
http://localhost:8080/api/products/health
```

Test the Order Service routing rule:

```bash
curl -i -H "Host: dev.localenv" \
http://localhost:8080/api/orders/123
```

The Order Service response depends on the endpoint and HTTP method supported by the application.

## Important Design Decisions

1. **Single reusable Helm chart:** Avoids maintaining duplicate deployment templates for DEV and QA.
2. **Environment-specific values:** Keeps environment differences separate from common application configuration.
3. **Namespace isolation:** DEV and QA have independent Kubernetes resources and Helm releases.
4. **Kubernetes Service discovery:** Applications communicate using Kubernetes Service DNS rather than Pod IP addresses.
5. **Secret references:** Database credentials are injected from Kubernetes Secrets.
6. **Ingress-based routing:** One existing NGINX Ingress Controller handles traffic for multiple application environments.
7. **Helm release history:** Supports deployment upgrades and rollback operations.

## Future Enhancements

- Integrate Helm deployments into GitHub Actions or Jenkins.
- Implement automated chart validation in CI pipelines.
- Package and publish Helm charts to an OCI registry.
- Add Helm chart dependencies and reusable subcharts.
- Introduce production-specific configuration.
- Integrate external secret management.
- Implement automated deployment verification and rollback strategies.
- Add resource requests, limits and application monitoring.

## Learning Outcomes

Through this project, I gained practical experience in:

- Developing reusable Helm charts.
- Deploying Spring Boot microservices using Helm.
- Managing DEV and QA environments with separate values files.
- Understanding Helm release revisions, upgrades and rollbacks.
- Integrating Kubernetes Services with Ingress routing.
- Configuring PostgreSQL connectivity using Kubernetes Secrets.
- Managing deployment configuration through Git.

---

**Project:** Product Order – Helm-Based Kubernetes Deployment

**Focus:** Helm, Kubernetes, Spring Boot Microservices, Environment Management, Ingress and Release Management.
