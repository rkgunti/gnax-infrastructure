# K8s Local Platform

Local Kubernetes platform running on Rancher Desktop with independent database and service releases.

## Prerequisites

Install and start Rancher Desktop with Kubernetes enabled. Install `kubectl` and Helm 3 or later.

Verify the tools and cluster context:

```bash
kubectl version --client
helm version
kubectl config current-context
kubectl get nodes
```

The selected context must point to the Rancher Desktop Kubernetes cluster.

## Repository Structure

```text
helm/
	databases/
		Chart.yaml
		values.yaml
		templates/
			mysql.yaml
			mongodb.yaml
	services/
		<service-name>/

environments/
	dev/
		databases.yaml
		databases.secret.yaml.example
		<service-name>.yaml
	test/
		databases.yaml
		databases.secret.yaml.example
		<service-name>.yaml
	uat/
		databases.yaml
		databases.secret.yaml.example
		<service-name>.yaml

namespaces/
	dev.yaml
	test.yaml
	uat.yaml
	dev-db.yaml
	test-db.yaml
	uat-db.yaml
```

Use generic service names such as `orders`, `catalog`, or `notifications`. Do not use architectural names such as `frontend` or `backend`.

## Namespace Model

Application services use `dev`, `test`, and `uat`. Databases use `dev-db`, `test-db`, and `uat-db`. One environment can contain many services while its databases remain isolated in the matching database namespace.

## Step 1: Create Namespaces

```bash
kubectl apply -f namespaces/
kubectl get namespaces
```

## Step 2: Prepare Local Database Secrets

Passwords are intentionally absent from tracked values files. Kubernetes `Secret` objects protect values inside the cluster, but their data is base64-encoded rather than automatically encrypted by this chart.

Create ignored secret override files:

```bash
cp environments/dev/databases.secret.yaml.example environments/dev/databases.secret.yaml
cp environments/test/databases.secret.yaml.example environments/test/databases.secret.yaml
cp environments/uat/databases.secret.yaml.example environments/uat/databases.secret.yaml
```

Open each new `databases.secret.yaml` file and replace every `CHANGE_ME_*` value with a local password. Never commit these files. They are ignored by `.gitignore`.

## Step 3: Validate the Database Chart

Lint the chart:

```bash
helm lint ./helm/databases
```

Render each environment using both its normal and secret values:

```bash
helm template databases ./helm/databases \
	--namespace dev-db \
	--values ./environments/dev/databases.yaml \
	--values ./environments/dev/databases.secret.yaml

helm template databases ./helm/databases \
	--namespace test-db \
	--values ./environments/test/databases.yaml \
	--values ./environments/test/databases.secret.yaml

helm template databases ./helm/databases \
	--namespace uat-db \
	--values ./environments/uat/databases.yaml \
	--values ./environments/uat/databases.secret.yaml
```

The chart intentionally fails if a secret values file is not supplied.

## Step 4: Deploy Databases

```bash
helm upgrade --install databases ./helm/databases \
	--namespace dev-db \
	--create-namespace \
	--values ./environments/dev/databases.yaml \
	--values ./environments/dev/databases.secret.yaml

helm upgrade --install databases ./helm/databases \
	--namespace test-db \
	--create-namespace \
	--values ./environments/test/databases.yaml \
	--values ./environments/test/databases.secret.yaml

helm upgrade --install databases ./helm/databases \
	--namespace uat-db \
	--create-namespace \
	--values ./environments/uat/databases.yaml \
	--values ./environments/uat/databases.secret.yaml
```

Check the database workloads:

```bash
kubectl get pods,svc,pvc -n dev-db
kubectl get pods,svc,pvc -n test-db
kubectl get pods,svc,pvc -n uat-db
helm list -A
```

## Step 5: Add a Service

For a service named `<service-name>`:

1. Create `helm/services/<service-name>/` with its own `Chart.yaml`, `values.yaml`, and templates.
2. Add environment overrides at `environments/dev/<service-name>.yaml`, `environments/test/<service-name>.yaml`, and `environments/uat/<service-name>.yaml` as needed.
3. Keep the service release in `dev`, `test`, or `uat`, never in a database namespace.
4. Use a `ClusterIP` Service for normal in-cluster traffic.
5. Add ingress or `NodePort` only when host or external access is required.

Validate the service chart:

```bash
helm lint ./helm/services/<service-name>
helm template <service-name> ./helm/services/<service-name> \
	--namespace dev \
	--values ./environments/dev/<service-name>.yaml
```

## Step 6: Deploy a Service

```bash
helm upgrade --install <service-name> ./helm/services/<service-name> \
	--namespace dev \
	--create-namespace \
	--values ./environments/dev/<service-name>.yaml
```

Use `test` or `uat` and the matching environment values for those deployments.

## Connection Details

Services inside the cluster use Kubernetes DNS:

```text
mongodb.dev-db.svc.cluster.local:27017
mysql.dev-db.svc.cluster.local:3306
```

Replace `dev-db` with `test-db` or `uat-db` for the other environments.

Host-side tools use these NodePorts:

```text
						 MongoDB    MySQL
dev          30017      30306
test         30018      30307
uat          30019      30308
```

Example local MongoDB connection:

```text
mongodb://<user>:<password>@127.0.0.1:30017/<database>
```

## Troubleshooting

```bash
kubectl get pods -A
kubectl describe pod <pod-name> -n <namespace>
kubectl logs deployment/mongodb -n dev-db
kubectl logs deployment/mysql -n dev-db
kubectl get endpoints -n dev-db
helm get manifest databases -n dev-db
```

Database PVCs are namespace-scoped. Do not move a database to another namespace without a backup and restore plan. For disposable local data, uninstall the old release and install the new one only when data loss is acceptable.

## Complete Cleanup

See [UNINSTALL.md](UNINSTALL.md) for the step-by-step procedure to remove service releases, database releases, PVCs, namespaces, and local secret files.

For team or CI usage, use an encrypted secret workflow such as SOPS with age or a secret manager such as Vault or External Secrets. Do not treat base64-encoded Kubernetes Secret manifests as encryption.