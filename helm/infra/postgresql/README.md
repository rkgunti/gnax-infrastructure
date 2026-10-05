# PostgreSQL

This chart deploys a single PostgreSQL instance as an independent Helm release
in a local infrastructure namespace. It is intended for Rancher Desktop
development, not production use.

## Current local settings

The release uses namespace `dev-infra`, database `devdb`, application user
`devuser`, and host NodePort `30432`. Its non-secret settings are in
`values/infra/postgresql.yaml`.

PostgreSQL listens on port `5432` in the cluster. The Service and Deployment
are named `postgresql`; the chart creates an `8Gi` PVC named `postgresql-pvc`.

## Prepare a password

Create the ignored secret values file and replace the
placeholder with a local password:

```bash
cp values/infra/postgresql.secret.yaml.example values/infra/postgresql.secret.yaml
```

The chart requires `postgresql.password`. The application user, database name,
image, storage, and Service settings come from
`values/infra/postgresql.yaml`.

## Install or update

Ensure namespace `dev-infra` exists, then install or update the release:

```bash
helm upgrade --install postgresql ./helm/infra/postgresql \
  --namespace dev-infra \
  --values values/infra/postgresql.yaml \
  --values values/infra/postgresql.secret.yaml
```

Validate the chart and check the workload:

```bash
helm lint ./helm/infra/postgresql --set postgresql.password=lint-only
kubectl get pods,svc -n dev-infra -l app=postgresql
kubectl get pvc postgresql-pvc -n dev-infra
kubectl logs deployment/postgresql -n dev-infra
```

## Connect

In-cluster clients use `postgresql.dev-infra.svc.cluster.local:5432`.
Host-side tools connect through `127.0.0.1:30432`. The local connection URL is
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

Uninstalling the release does not remove local secret override files.
