# Three-Tier EKS GitOps

Kubernetes desired-state configuration continuously reconciled to Amazon EKS by Argo CD.

## Architecture

```text
Internet
   |
AWS Application Load Balancer
   |
Frontend (Nginx)
   |
Backend API
   |        |
PostgreSQL Redis
```

## Repository structure

```text
apps/dev/
  kustomization.yaml
  stack.yaml
  monitoring.yaml
  three-tier-dashboard.json

bootstrap/dev/
  argocd.yaml
```

* `apps/dev` contains the development Kubernetes resources.
* `bootstrap/dev` contains the Argo CD project and application.

## Development stack

The development environment deploys:

* Three frontend replicas
* Two backend replicas
* One PostgreSQL replica with persistent storage
* One Redis replica with ephemeral storage
* Internal Kubernetes services
* An AWS Load Balancer Controller Ingress
* Health and readiness probes
* CPU and memory requests and limits

## Container images

The application uses commit-specific images published to GitHub Container Registry:

```text
ghcr.io/techworld707/three-tier-backend
ghcr.io/techworld707/three-tier-frontend
```

The manifests pin immutable commit-SHA tags instead of relying on `latest`.

## GitOps behavior

Argo CD monitors `apps/dev` on the `main` branch and automatically:

* Applies desired-state changes
* Repairs configuration drift
* Prunes removed resources
* Creates the application namespace

## Validation

Render the development configuration:

```bash
kubectl kustomize apps/dev
```

Validate the Argo CD bootstrap resources locally:

```bash
kubectl apply \
  --dry-run=client \
  -f bootstrap/dev/argocd.yaml
```

Check repository formatting:

```bash
git diff --check
```

## Deployment

The AWS infrastructure and EKS cluster must exist before deployment.

Configure Kubernetes access:

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name three-tier-eks-dev \
  --profile three-tier-admin
```

Bootstrap the Argo CD application:

```bash
kubectl apply -f bootstrap/dev/argocd.yaml
```

Argo CD will then deploy and reconcile the resources under `apps/dev`.

## Observability rollout

1. Apply the platform repository's development add-ons before this GitOps change
   reaches `main`. Confirm the monitoring services and Prometheus Operator CRDs
   exist with `kubectl -n monitoring get pods,svc` and
   `kubectl get crd servicemonitors.monitoring.coreos.com probes.monitoring.coreos.com prometheusrules.monitoring.coreos.com`.
2. Publish the updated frontend and backend images from the application repository.
   Update both image tags in `apps/dev/stack.yaml` to the published commit SHA.
3. Merge the GitOps changes and confirm `three-tier-dev` is synced and healthy in
   Argo CD. Check the backend scrape and frontend probe targets in Prometheus.

Open Grafana locally:

```bash
kubectl -n monitoring port-forward svc/monitoring-grafana 3000:80
```

Use the `admin-user` and `admin-password` keys in the `monitoring-grafana` Secret
to sign in at `http://localhost:3000`. Open **Three-tier development**. Set the ALB
and target group filters to their ARN suffixes for app-specific CloudWatch panels.
The HTTP availability probe measures the internal frontend Service. The database
panels use SQLAlchemy pool metrics for the PostgreSQL instance inside EKS.

Generate traffic through the frontend by submitting test data. Check the Prometheus
panels, then use CloudWatch Logs Insights on
`/aws/eks/<cluster-name>/three-tier-dev` to inspect the request:

```text
fields @timestamp, kubernetes.container_name, request_id, trace_id, status, message
| filter request_id = "<X-Request-ID from the API response>"
| sort @timestamp asc
```

For a sampled request, open Grafana Explore with the Tempo data source and look up
the backend log's `trace_id` to see the Flask and PostgreSQL spans. Use the
`/aws/eks/<cluster-name>/cluster` log group for control-plane events.

## Development limitations

This configuration intentionally prioritizes a simple development setup:

* PostgreSQL runs inside Kubernetes instead of Amazon RDS.
* Redis runs inside Kubernetes instead of Amazon ElastiCache.
* Redis data is ephemeral.
* The database credential is development-only.
* The Ingress does not configure a custom domain or TLS certificate.

Production deployments should use managed data services, external secret management, TLS, backups, network policies, and a disaster-recovery plan.

## Related repositories

* Application: `three-tier-eks-application`
* Infrastructure: `terraform-aws-eks-gitops-platform`
* GitOps: `three-tier-eks-gitops`
