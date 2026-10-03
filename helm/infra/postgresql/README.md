# PostgreSQL

This chart deploys a single PostgreSQL instance as an independent Helm release
in a local infrastructure namespace. It is intended for Rancher Desktop
development, not production use.

## Per-environment settings

| Environment | Namespace | Database | User | Host NodePort |
| --- | --- | --- | --- | --- |
| dev | `dev-infra` | `devdb` | `devuser` | `30432` |
| test | `test-infra` | `testdb` | `testuser` | `30433` |
| uat | `uat-infra` | `uatdb` | `uatuser` | `30434` |

PostgreSQL listens on port `5432` in the cluster. The Service and Deployment
are named `postgresql`; the chart creates an `8Gi` PVC named `postgresql-pvc`.

## Prepare a password

Create the ignored secret values file for the environment and replace the
placeholder with a local password:

```bash
cp environments/dev/postgresql.secret.yaml.example environments/dev/postgresql.secret.yaml
```

The chart requires `postgresql.password`. The application user, database name,
image, storage, and Service settings come from
`environments/<env>/postgresql.yaml`.

## Install or update

Ensure the environment namespaces have been created with `kubectl apply -f namespaces/`.
For dev:

```bash
helm upgrade --install postgresql ./helm/infra/postgresql \
  --namespace dev-infra \
  --values environments/dev/postgresql.yaml \
  --values environments/dev/postgresql.secret.yaml
```

For test or uat, use the matching environment name in the namespace and both
values-file paths. Each environment is a separate release in its own namespace.

Validate the chart and check the workload:

```bash
helm lint ./helm/infra/postgresql --set postgresql.password=lint-only
kubectl get pods,svc -n dev-infra -l app=postgresql
kubectl get pvc postgresql-pvc -n dev-infra
kubectl logs deployment/postgresql -n dev-infra
```

## Connect

In-cluster clients use `postgresql.<env>-infra.svc.cluster.local:5432`, for
example `postgresql.dev-infra.svc.cluster.local:5432`. Host-side tools connect
through `127.0.0.1:30432` for dev, `127.0.0.1:30433` for test, or
`127.0.0.1:30434` for uat. The dev connection URL is
`postgresql://devuser:<password>@127.0.0.1:30432/devdb`.

## Uninstall

Check the release and storage first:

```bash
helm list --namespace dev-infra
kubectl get pvc -n dev-infra
```

Back up any needed data before uninstalling. The release manages the
`postgresql-pvc`; Helm normally removes that PVC along with its Deployment,
Service, and Secret. Underlying data reclamation depends on the storage class
and its reclaim policy.

```bash
helm uninstall postgresql --namespace dev-infra
```

For test or uat, use the corresponding `test-infra` or `uat-infra` namespace.
Uninstalling the release does not remove local secret override files.
