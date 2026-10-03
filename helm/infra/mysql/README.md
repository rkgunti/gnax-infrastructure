# MySQL

This chart deploys a single MySQL instance as an independent Helm release in a
local infrastructure namespace. It is intended for Rancher Desktop development,
not production use.

## Per-environment settings

| Environment | Namespace | Database | User | Host NodePort |
| --- | --- | --- | --- | --- |
| dev | `dev-infra` | `devdb` | `devuser` | `30306` |
| test | `test-infra` | `testdb` | `testuser` | `30307` |
| uat | `uat-infra` | `uatdb` | `uatuser` | `30308` |

MySQL listens on port `3306` in the cluster. The Service and Deployment are
named `mysql`; the chart creates an `8Gi` PVC named `mysql-pvc`.

## Prepare a password

Create the ignored secret values file for the environment and replace both
placeholders with local passwords:

```bash
cp environments/dev/mysql.secret.yaml.example environments/dev/mysql.secret.yaml
```

The chart requires `mysql.rootPassword` and `mysql.password`. The application
user, database name, image, storage, and Service settings come from
`environments/<env>/mysql.yaml`.

## Install or update

Ensure the environment namespaces have been created with `kubectl apply -f namespaces/`.
For dev:

```bash
helm upgrade --install mysql ./helm/infra/mysql \
  --namespace dev-infra \
  --values environments/dev/mysql.yaml \
  --values environments/dev/mysql.secret.yaml
```

For test or uat, use the matching environment name in the namespace and both
values-file paths. Each environment is a separate release in its own namespace.

Validate the chart and check the workload:

```bash
helm lint ./helm/infra/mysql --set mysql.rootPassword=lint-only --set mysql.password=lint-only
kubectl get pods,svc -n dev-infra -l app=mysql
kubectl get pvc mysql-pvc -n dev-infra
kubectl logs deployment/mysql -n dev-infra
```

## Connect

In-cluster clients use `mysql.<env>-infra.svc.cluster.local:3306`, for example
`mysql.dev-infra.svc.cluster.local:3306`. Host-side tools connect through
`127.0.0.1:30306` for dev, `127.0.0.1:30307` for test, or `127.0.0.1:30308`
for uat. Use the application credentials in the local secret file.

## Uninstall

Check the release and storage first:

```bash
helm list --namespace dev-infra
kubectl get pvc -n dev-infra
```

Back up any needed data before uninstalling. The release manages the
`mysql-pvc`; Helm normally removes that PVC along with its Deployment, Service,
and Secret. Underlying data reclamation depends on the storage class and its
reclaim policy.

```bash
helm uninstall mysql --namespace dev-infra
```

For test or uat, use the corresponding `test-infra` or `uat-infra` namespace.
Uninstalling the release does not remove local secret override files.
