# MongoDB

This chart deploys a single MongoDB instance as an independent Helm release in a
local infrastructure namespace. It is intended for Rancher Desktop development,
not production use.

## Current local settings

The release uses namespace `dev-infra`, database `devdb`, application user
`devuser`, and host NodePort `30017`. Its non-secret settings are in
`values/infra/mongodb.yaml`.

MongoDB listens on port `27017` in the cluster. The Service and Deployment are
named `mongodb`; the chart creates a `10Gi` PVC named `mongo-pvc`. It creates a
root user and a database-scoped application user.

## Prepare passwords

Create the ignored secret values file and replace both
placeholders with local passwords:

```bash
cp values/infra/mongodb.secret.yaml.example values/infra/mongodb.secret.yaml
```

The chart requires `mongodb.rootPassword` and `mongodb.password`. The root
username, application username, database name, image, storage, and Service
settings come from `values/infra/mongodb.yaml`.

## Install or update

Ensure namespace `dev-infra` exists, then install or update the release:

```bash
helm upgrade --install mongodb ./helm/infra/mongodb \
  --namespace dev-infra \
  --values values/infra/mongodb.yaml \
  --values values/infra/mongodb.secret.yaml
```

Validate the chart and check the workload:

```bash
helm lint ./helm/infra/mongodb --set mongodb.rootPassword=lint-only --set mongodb.password=lint-only
kubectl get pods,svc -n dev-infra -l app=mongodb
kubectl get pvc mongo-pvc -n dev-infra
kubectl logs deployment/mongodb -n dev-infra
```

## Connect

In-cluster clients use `mongodb.dev-infra.svc.cluster.local:27017`; host-side
tools connect through `127.0.0.1:30017`. Use the application credentials from
the local secret file and authenticate against the configured application
database.

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

Uninstalling the release does not remove local secret override files.
