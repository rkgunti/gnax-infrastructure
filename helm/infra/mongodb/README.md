# MongoDB

This chart deploys a single MongoDB instance as an independent Helm release in a
local infrastructure namespace. It is intended for Rancher Desktop development,
not production use.

## Per-environment settings

| Environment | Namespace | Database | Application user | Host NodePort |
| --- | --- | --- | --- | --- |
| dev | `dev-infra` | `devdb` | `devuser` | `30017` |
| test | `test-infra` | `testdb` | `testuser` | `30018` |
| uat | `uat-infra` | `uatdb` | `uatuser` | `30019` |

MongoDB listens on port `27017` in the cluster. The Service and Deployment are
named `mongodb`; the chart creates a `10Gi` PVC named `mongo-pvc`. It creates a
root user and a database-scoped application user.

## Prepare passwords

Create the ignored secret values file for the environment and replace both
placeholders with local passwords:

```bash
cp environments/dev/mongodb.secret.yaml.example environments/dev/mongodb.secret.yaml
```

The chart requires `mongodb.rootPassword` and `mongodb.password`. The root
username, application username, database name, image, storage, and Service
settings come from `environments/<env>/mongodb.yaml`.

## Install or update

Ensure the environment namespaces have been created with `kubectl apply -f namespaces/`.
For dev:

```bash
helm upgrade --install mongodb ./helm/infra/mongodb \
  --namespace dev-infra \
  --values environments/dev/mongodb.yaml \
  --values environments/dev/mongodb.secret.yaml
```

For test or uat, use the matching environment name in the namespace and both
values-file paths. Each environment is a separate release in its own namespace.

Validate the chart and check the workload:

```bash
helm lint ./helm/infra/mongodb --set mongodb.rootPassword=lint-only --set mongodb.password=lint-only
kubectl get pods,svc -n dev-infra -l app=mongodb
kubectl get pvc mongo-pvc -n dev-infra
kubectl logs deployment/mongodb -n dev-infra
```

## Connect

In-cluster clients use `mongodb.<env>-infra.svc.cluster.local:27017`, for
example `mongodb.dev-infra.svc.cluster.local:27017`. Host-side tools connect
through `127.0.0.1:30017` for dev, `127.0.0.1:30018` for test, or
`127.0.0.1:30019` for uat. Use the application credentials from the local
secret file and authenticate against the configured application database.

## Uninstall

Check the release and storage first:

```bash
helm list --namespace dev-infra
kubectl get pvc -n dev-infra
```

Back up any needed data before uninstalling. The release manages the
`mongo-pvc`; Helm normally removes that PVC along with its Deployment, Service,
Secret, and initialization ConfigMap. Underlying data reclamation depends on
the storage class and its reclaim policy.

```bash
helm uninstall mongodb --namespace dev-infra
```

For test or uat, use the corresponding `test-infra` or `uat-infra` namespace.
Uninstalling the release does not remove local secret override files.
