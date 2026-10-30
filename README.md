# PickAndOrder GitOps repository

This repository is the desired-state source for Kubernetes deployments. The application source repository builds and tests the Product Service image, pushes it to a container registry, then opens a commit/PR here to update the image tag. Argo CD continuously reconciles Kubernetes to the committed manifests.

## Design

```text
Application source repo
  └─ GitHub Actions: test → build image → scan → push image
       └─ update image.tag in this GitOps repo (prefer a PR)
            └─ merge to main
                 └─ Argo CD ApplicationSet detects Git change
                      └─ Helm chart + environment values
                           └─ Kubernetes Deployment / Service / ALB Ingress
```

- `charts/product-service/`: versioned Helm chart for the service.
- `clusters/dev/apps/product-service/values.yaml`: dev-specific desired state.
- `clusters/prod/apps/product-service/values.yaml`: production-specific desired state.
- `argocd/applicationsets/`: ApplicationSet definitions. Install the environment-specific ApplicationSet into the corresponding cluster's Argo CD.
- Neon PostgreSQL and Aiven Kafka remain managed services. This repository does not deploy either database or broker.
- AWS Load Balancer Controller provisions an ALB from the chart's Ingress. Install and configure the controller, IAM permissions, subnet discovery, and ACM certificate separately in each EKS cluster.

## First-time setup

### 1. Create the GitHub repository

Create a private GitHub repository, for example `pickandorder-gitops`, then push this repository's contents to its `main` branch. Replace `YOUR_ORG` in the ApplicationSet files with the actual GitHub organization/user and repository URL.

### 2. Build and push the image from the source repository

Build the image from the application repository and push it to ECR (recommended for EKS) or your selected registry. The chart values use an image repository and tag; the CI workflow should update only the environment's `image.tag`, ideally via a pull request. Do not let application CI directly change cluster state.

Example image value:
```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/pickandorder-product-service
  tag: "git-sha-or-release-tag"
```

Use immutable tags such as a commit SHA or release version. Avoid `latest` in GitOps because it is not a reliable deployment identifier.

### 3. Create Kubernetes secrets

Create secrets in the target cluster and namespace, preferably through External Secrets Operator with AWS Secrets Manager or another approved secret manager. Do not commit real credentials or certificate files here.

The chart expects an application Secret named `product-service-secrets` with keys:
- `DATABASE_URL` (Neon pooled runtime URL)
- `DIRECT_URL` (Neon direct URL, if required by Prisma)
- `KAFKA_USERNAME`
- `KAFKA_PASSWORD`

It also expects a Secret named `product-service-aiven-ca` with key `aiven-ca.pem` containing the Aiven CA certificate. For example, for local/dev bootstrap only:
```bash
kubectl create namespace pickandorder
kubectl -n pickandorder create secret generic product-service-secrets \
  --from-literal=DATABASE_URL="$DATABASE_URL" \
  --from-literal=DIRECT_URL="$DIRECT_URL" \
  --from-literal=KAFKA_USERNAME="$KAFKA_USERNAME" \
  --from-literal=KAFKA_PASSWORD="$KAFKA_PASSWORD"
kubectl -n pickandorder create secret generic product-service-aiven-ca \
  --from-file=aiven-ca.pem=./aiven-ca.pem
```

Kubernetes Secret values are not automatically encrypted just because they are stored as Secret objects. Enable encryption at rest, restrict RBAC, and prefer an external secret manager for production.

### 4. Configure the AWS ALB

In the environment's `values.yaml`, set:
- `ingress.enabled: true`
- `ingress.className: alb`
- `ingress.annotations.alb.ingress.kubernetes.io/scheme`: `internet-facing` or `internal`
- `ingress.annotations.alb.ingress.kubernetes.io/target-type`: `ip`
- `ingress.annotations.alb.ingress.kubernetes.io/certificate-arn`: your ACM certificate ARN
- listener ports and health-check path as appropriate.

The chart defaults to `ingress.enabled: false`. For internet-facing deployments, use HTTPS and a valid ACM certificate. The AWS Load Balancer Controller must be installed in EKS and have the appropriate IAM role/service account and subnet discovery tags. The ALB controller and Argo CD installation are cluster prerequisites, not managed by this chart.

### 5. Install Argo CD ApplicationSet

This repository assumes one Argo CD installation per cluster, with the Argo CD local cluster as the destination. From the cluster where Argo CD is installed, apply the matching ApplicationSet after replacing the repository URL:

```bash
kubectl apply -f argocd/applicationsets/dev.yaml
# In the production cluster instead:
kubectl apply -f argocd/applicationsets/prod.yaml
```

The ApplicationSet creates an Argo CD Application for each directory under the corresponding `clusters/<env>/apps/*`. It uses multi-source Applications (Argo CD 2.6+) so the chart is in `charts/` and environment values remain in `clusters/`.

For a private GitHub repository, configure repository credentials in Argo CD (for example, a GitHub App or deploy key) rather than embedding a token in YAML.

### 6. Validate and observe

```bash
helm lint charts/product-service
helm template product-service charts/product-service \
  --namespace pickandorder \
  -f clusters/dev/apps/product-service/values.yaml

kubectl -n argocd get applicationsets
kubectl -n argocd get applications
kubectl -n pickandorder get deploy,svc,ingress
```

Argo CD auto-sync is enabled with prune and self-heal. A merged Git change is the deployment approval point. For production, protect `main`, require PR review, and consider a manual approval or promotion PR from dev to prod.

## Adding a future service

1. Add its chart at `charts/<service-name>/`.
2. Add environment values at `clusters/dev/apps/<service-name>/values.yaml` and/or `clusters/prod/apps/<service-name>/values.yaml`.
3. Ensure the matching ApplicationSet is watching that environment directory. It will discover the new service automatically; no ApplicationSet code change is required.
4. Add the service's secret contract and ALB/DNS configuration as required.

The generic ApplicationSet expects each chart to use its directory name and each values directory to contain `values.yaml`. If a future service needs a different chart layout, adapt its chart or introduce a small per-service exception rather than duplicating the whole deployment mechanism.

## CI image update contract

The application repository's GitHub Actions should:
1. Test and build.
2. Scan and push an immutable image tag.
3. Update `image.tag` only in the target environment values file.
4. Open a PR against this GitOps repo (or push to a protected automation branch and create a PR).
5. Let Argo CD deploy only after that change is merged.

Use a narrowly scoped GitHub App or fine-grained token for the GitOps PR, and never expose production secrets to untrusted pull-request workflows.
