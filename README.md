# EKS deployment manifests

These manifests deploy the backend services to the `fiveline` namespace.

## Files

- `namespace.yaml`: creates the `fiveline` namespace.
- `configmap.yaml`: non-sensitive runtime values shared by the services.
- `secret.example.yaml`: template for sensitive values. Copy it to `secret.yaml` and fill real values locally.
- `user-service.yaml`: `Deployment` and `Service` for auth/user APIs.
- `product-service.yaml`: `Deployment` and `Service` for product/review APIs.
- `order-service.yaml`: `Deployment` and `Service` for cart/order APIs.
- `ingress.yaml`: ALB Ingress routing `/api/*` paths to the right service.
- `kustomization.yaml`: applies all deployment resources together.

## Before applying

Create the real secret file locally:

```bash
cp k8s/secret.example.yaml k8s/secret.yaml
```

Then edit `k8s/secret.yaml`:

```yaml
DATABASE_URL: postgresql+psycopg://postgres:<RDS_PASSWORD>@fiveline.chsgo2iu4ob1.ap-northeast-2.rds.amazonaws.com:5432/aiops?sslmode=require
JWT_SECRET: <LONG_RANDOM_VALUE>
```

Do not commit `k8s/secret.yaml`.

## Apply

```bash
kubectl apply -k k8s
kubectl get pods -n fiveline
kubectl get ingress -n fiveline
```

If the Ingress address does not appear, check that AWS Load Balancer Controller is installed in the EKS cluster.

