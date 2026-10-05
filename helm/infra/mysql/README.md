# MySQL

This chart deploys a single MySQL instance as an independent Helm release in a
local infrastructure namespace. It is intended for Rancher Desktop development,
not production use.

## Current local settings

The release uses namespace `dev-infra`, database `devdb`, application user
`devuser`, and host NodePort `30306`. Its non-secret settings are in
`values/infra/mysql.yaml`.

MySQL listens on port `3306` in the cluster. The Service and Deployment are
named `mysql`; the chart creates an `8Gi` PVC named `mysql-pvc`.

## Prepare a password

Create the ignored secret values file and replace both
placeholders with local passwords:

```bash
cp values/infra/mysql.secret.yaml.example values/infra/mysql.secret.yaml
```

The chart requires `mysql.rootPassword` and `mysql.password`. The application
user, database name, image, storage, and Service settings come from
`values/infra/mysql.yaml`.

## Install or update

Ensure namespace `dev-infra` exists, then install or update the release:

```bash
helm upgrade --install mysql ./helm/infra/mysql \
  --namespace dev-infra \
  --values values/infra/mysql.yaml \
  --values values/infra/mysql.secret.yaml
```

Validate the chart and check the workload:

```bash
helm lint ./helm/infra/mysql --set mysql.rootPassword=lint-only --set mysql.password=lint-only
kubectl get pods,svc -n dev-infra -l app=mysql
kubectl get pvc mysql-pvc -n dev-infra
kubectl logs deployment/mysql -n dev-infra
```

## Connect

In-cluster clients use `mysql.dev-infra.svc.cluster.local:3306`; host-side
tools connect through `127.0.0.1:30306`. Use the application credentials in
the local secret file.

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

Uninstalling the release does not remove local secret override files.
