# gnax-infrastructure
Infrastructure and deployment configurations for GnaX, including Kubernetes, Helm, Kafka, databases, Redis, networking, and environment configurations.
Local Kubernetes infrastructure running on Rancher Desktop with independent database and service releases.

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
	infra/
		mysql/
		mongodb/
		postgresql/
		kafka/
	services/
		<service-name>/

environments/
	dev/
		mysql.yaml
		mysql.secret.yaml.example
		mongodb.yaml
		mongodb.secret.yaml.example
		postgresql.yaml
		postgresql.secret.yaml.example
		kafka.yaml
		<service-name>.yaml
	test/
		mysql.yaml
		mysql.secret.yaml.example
		mongodb.yaml
		mongodb.secret.yaml.example
		postgresql.yaml
		postgresql.secret.yaml.example
		kafka.yaml
		<service-name>.yaml
	uat/
		mysql.yaml
		mysql.secret.yaml.example
		mongodb.yaml
		mongodb.secret.yaml.example
		postgresql.yaml
		postgresql.secret.yaml.example
		kafka.yaml
		<service-name>.yaml

namespaces/
	dev.yaml
	test.yaml
	uat.yaml
	dev-infra.yaml
	test-infra.yaml
	uat-infra.yaml
```

Use generic service names such as `orders`, `catalog`, or `notifications`. Do not use architectural names such as `frontend` or `backend`.

## Namespace Model

Application services use `dev`, `test`, and `uat`. Infrastructure components use `dev-infra`, `test-infra`, and `uat-infra`. One environment can contain many services while its infrastructure components remain isolated in the matching infrastructure namespace.

Each chart's README contains install, operation, and uninstall instructions:

- [MySQL](./helm/infra/mysql/README.md)
- [MongoDB](./helm/infra/mongodb/README.md)
- [PostgreSQL](./helm/infra/postgresql/README.md)
- [Kafka](./helm/infra/kafka/README.md)
- [GnaX Config Server service and Secret](./helm/services/gnax-config-server/README.md)

## Step 1: Create Namespaces

```bash
kubectl apply -f namespaces/
kubectl get namespaces
```

## Step 2: Prepare Local Database Secrets

Passwords are intentionally absent from tracked values files. Kubernetes `Secret` objects protect values inside the cluster, but their data is base64-encoded rather than automatically encrypted by this chart.

Create ignored secret override files:

```bash
cp environments/dev/mysql.secret.yaml.example environments/dev/mysql.secret.yaml
cp environments/dev/mongodb.secret.yaml.example environments/dev/mongodb.secret.yaml
cp environments/dev/postgresql.secret.yaml.example environments/dev/postgresql.secret.yaml
cp environments/test/mysql.secret.yaml.example environments/test/mysql.secret.yaml
cp environments/test/mongodb.secret.yaml.example environments/test/mongodb.secret.yaml
cp environments/test/postgresql.secret.yaml.example environments/test/postgresql.secret.yaml
cp environments/uat/mysql.secret.yaml.example environments/uat/mysql.secret.yaml
cp environments/uat/mongodb.secret.yaml.example environments/uat/mongodb.secret.yaml
cp environments/uat/postgresql.secret.yaml.example environments/uat/postgresql.secret.yaml
for e in dev test uat; do cp environments/$e/gnax-config-server.secret.yaml.example environments/$e/gnax-config-server.secret.yaml; done
```

Open each new database secret file and the `gnax-config-server.secret.yaml` files,
then replace every `CHANGE_ME_*` value with the appropriate local value.
Never commit these files; they are ignored by `.gitignore`.

## Step 3: Validate the Infrastructure Charts

Lint each independent chart:

```bash
helm lint ./helm/infra/mysql --set mysql.rootPassword=lint-only --set mysql.password=lint-only
helm lint ./helm/infra/mongodb --set mongodb.rootPassword=lint-only --set mongodb.password=lint-only
helm lint ./helm/infra/postgresql --set postgresql.password=lint-only
helm lint ./helm/infra/kafka
helm lint ./helm/services/gnax-config-server \
	--values ./environments/dev/gnax-config-server.yaml \
	--values ./environments/dev/gnax-config-server.secret.yaml.example
```

Render each chart independently. Database and Config Server secret charts
require their matching local secret override file:

```bash
helm template mysql ./helm/infra/mysql \
	--namespace dev-infra \
	--values ./environments/dev/mysql.yaml \
	--values ./environments/dev/mysql.secret.yaml

helm template mongodb ./helm/infra/mongodb \
	--namespace dev-infra \
	--values ./environments/dev/mongodb.yaml \
	--values ./environments/dev/mongodb.secret.yaml

helm template postgresql ./helm/infra/postgresql \
	--namespace dev-infra \
	--values ./environments/dev/postgresql.yaml \
	--values ./environments/dev/postgresql.secret.yaml

helm template kafka ./helm/infra/kafka \
	--namespace dev-infra \
	--values ./environments/dev/kafka.yaml

helm template gnax-config-server ./helm/services/gnax-config-server \
	--namespace dev \
	--values ./environments/dev/gnax-config-server.yaml \
	--values ./environments/dev/gnax-config-server.secret.yaml
```

## Step 4: Deploy Databases Independently

Deploy or update MySQL:

```bash
helm upgrade --install mysql ./helm/infra/mysql \
	--namespace dev-infra \
	--create-namespace \
	--values ./environments/dev/mysql.yaml \
	--values ./environments/dev/mysql.secret.yaml
```

Deploy or update MongoDB:

```bash
helm upgrade --install mongodb ./helm/infra/mongodb \
	--namespace dev-infra \
	--create-namespace \
	--values ./environments/dev/mongodb.yaml \
	--values ./environments/dev/mongodb.secret.yaml
```

Deploy or update PostgreSQL:

```bash
helm upgrade --install postgresql ./helm/infra/postgresql \
	--namespace dev-infra \
	--create-namespace \
	--values ./environments/dev/postgresql.yaml \
	--values ./environments/dev/postgresql.secret.yaml
```

Repeat the database release commands with the matching environment values and namespace to deploy independently into `test-infra` or `uat-infra`.

The Config Server's Secret (`config-server-secret`) is created by the
`gnax-config-server` chart together with its Deployment and Service. Supply
`environments/dev/gnax-config-server.secret.yaml` when deploying the service below.
If an existing cluster still has a separate Helm release named `config-server`
in `dev`, follow the one-time migration instructions in the
[Config Server README](./helm/services/gnax-config-server/README.md) before
upgrading; the existing release must relinquish the Secret first.

Deploy or update Kafka:

```bash
helm upgrade --install kafka ./helm/infra/kafka \
	--namespace dev-infra \
	--create-namespace \
	--values ./environments/dev/kafka.yaml
```

Use the matching `test` or `uat` values files and namespaces for those environments. Check the independent releases and workloads:

```bash
helm list -n dev-infra
kubectl get pods,svc,pvc -n dev-infra
```

Kafka uses `kafka.dev-infra.svc.cluster.local:9092` inside Kubernetes and `127.0.0.1:30094` from host-side tools.

## Step 6: Add a Service

For a service named `<service-name>`:

1. Create `helm/services/<service-name>/` with its own `Chart.yaml`, `values.yaml`, and templates.
2. Add environment overrides at `environments/dev/<service-name>.yaml`, `environments/test/<service-name>.yaml`, and `environments/uat/<service-name>.yaml` as needed.
3. Keep the service release in `dev`, `test`, or `uat`, never in a database namespace.
4. Use a `ClusterIP` Service for normal in-cluster traffic.
5. Add ingress or `NodePort` only when host or external access is required.
6. Add a `README.md` beside the chart with its configuration, install, verification, and uninstall instructions. Document any separate Secrets, ConfigMaps, PVCs, or other resources and their data-retention behavior.

Validate the service chart:

```bash
helm lint ./helm/services/<service-name>
helm template <service-name> ./helm/services/<service-name> \
	--namespace dev \
	--values ./environments/dev/<service-name>.yaml
```

## Step 7: Deploy a Service

```bash
helm upgrade --install <service-name> ./helm/services/<service-name> \
	--namespace dev \
	--create-namespace \
	--values ./environments/dev/<service-name>.yaml
```

Use `test` or `uat` and the matching environment values for those deployments.

## Deploy the Config Server (dev)

The config server and its `config-server-secret` are managed together by the
`gnax-config-server` Helm release in the `dev` namespace (Kubernetes
Deployment and Service name: `config-server`).

1. Create the secret file and set the credentials and Git URI (the file is git-ignored):

```bash
cp environments/dev/gnax-config-server.secret.yaml.example environments/dev/gnax-config-server.secret.yaml
```

2. Build the image so the local cluster can use it (Rancher Desktop with dockerd; for containerd use `nerdctl --namespace k8s.io build`):

```bash
docker build -t gnax-config-server:0.0.1-SNAPSHOT ../gnax-config-server
```

3. Validate the chart and render the service resources, including its Secret:

```bash
helm lint ./helm/services/gnax-config-server \
	--values ./environments/dev/gnax-config-server.yaml \
	--values ./environments/dev/gnax-config-server.secret.yaml
helm template gnax-config-server ./helm/services/gnax-config-server \
	--namespace dev \
	--values ./environments/dev/gnax-config-server.yaml \
	--values ./environments/dev/gnax-config-server.secret.yaml
```

4. Deploy the application and its Secret in one Helm release, then wait for it:

```bash
helm upgrade --install gnax-config-server ./helm/services/gnax-config-server \
	--namespace dev \
	--values ./environments/dev/gnax-config-server.yaml \
	--values ./environments/dev/gnax-config-server.secret.yaml
kubectl -n dev rollout status deployment/config-server
kubectl -n dev get secret config-server-secret
```

5. Verify (use the credentials from the secret file):

```bash
kubectl -n dev get pods,svc -l app.kubernetes.io/name=config-server
kubectl -n dev port-forward svc/config-server 8888:8888
curl -u <username>:<password> http://localhost:8888/actuator/health
```

In-cluster URL for other services: `http://config-server.dev.svc.cluster.local:8888` (basic auth from the secret).

To change config or credentials, rerun the Helm upgrade with both values files
and restart the Deployment so it reloads the updated Secret:

```bash
kubectl -n dev rollout restart deployment/config-server
kubectl -n dev rollout status deployment/config-server
```

After code changes, rebuild the image (step 2, optionally with a new tag in
`environments/dev/gnax-config-server.yaml`) and rerun the upgrade. If the Pod
fails to start, check `kubectl -n dev logs deploy/config-server`.

## Connection Details

Services inside the cluster use Kubernetes DNS:

```text
mongodb.dev-infra.svc.cluster.local:27017
mysql.dev-infra.svc.cluster.local:3306
postgresql.dev-infra.svc.cluster.local:5432
kafka.dev-infra.svc.cluster.local:9092
```

Replace `dev-infra` with `test-infra` or `uat-infra` for the other environments.

Host-side tools use these NodePorts:

```text
Environment  MongoDB    MySQL       PostgreSQL  Kafka
dev          30017      30306      30432       30094
test         30018      30307      30433       30094
uat          30019      30308      30434       30094
```

Kafka clients inside the cluster use `kafka.<env>-infra.svc.cluster.local:9092`. Host-side tools use `127.0.0.1:30094`.

Example local MongoDB connection:

```text
mongodb://<user>:<password>@127.0.0.1:30017/<database>
```

Example local PostgreSQL connection:

```text
postgresql://devuser:<password>@127.0.0.1:30432/devdb
```

## Troubleshooting

```bash
kubectl get pods -A
kubectl describe pod <pod-name> -n <namespace>
kubectl logs deployment/mongodb -n dev-infra
kubectl logs deployment/mysql -n dev-infra
kubectl logs deployment/postgresql -n dev-infra
kubectl logs deployment/kafka -n dev-infra
kubectl get endpoints -n dev-infra
helm get manifest mysql -n dev-infra
helm get manifest mongodb -n dev-infra
helm get manifest postgresql -n dev-infra
helm get manifest kafka -n dev-infra
```

Database and Kafka PVCs are namespace-scoped. Do not move persistent data to
another namespace without a backup and restore plan. For disposable local
data, uninstall the old release and install the new one only when data loss is
acceptable.

## Complete Cleanup

These steps remove repository-managed services and infrastructure from the
currently selected Kubernetes cluster. They can delete Kubernetes Secrets,
ConfigMaps, workloads (Deployments/Pods), Services, PVCs, namespaces, and
database or Kafka data. Confirm the selected Rancher Desktop context and back
up any data you need before proceeding.

### 1. Inventory the cluster

```bash
kubectl config current-context
kubectl get nodes
helm list -A
kubectl get namespaces
kubectl get all,secret,configmap,pvc -n dev
kubectl get all,secret,configmap,pvc -n test
kubectl get all,secret,configmap,pvc -n uat
kubectl get all,secret,configmap,pvc -n dev-infra
kubectl get all,secret,configmap,pvc -n test-infra
kubectl get all,secret,configmap,pvc -n uat-infra
```

Review the release names and resources before removal. Do not remove releases,
Secrets, PVCs, or namespaces managed by other projects.

### 2. Remove application releases

The GnaX Config Server is installed as release `gnax-config-server` in `dev`:

```bash
helm uninstall gnax-config-server --namespace dev
```

For any other application releases, list each application namespace and
uninstall only the releases belonging to this project. Replace
`<release-name>` with the exact name shown by `helm list`:

```bash
helm list --namespace dev
helm uninstall <release-name> --namespace dev
helm list --namespace test
helm uninstall <release-name> --namespace test
helm list --namespace uat
helm uninstall <release-name> --namespace uat
```

Helm removes the release-managed Deployments, Pods, Services, and other
resources. Check for manually created resources separately.

The `gnax-config-server` release also manages the Config Server Kubernetes
Secret. Uninstalling the application release removes both together; there is
no separate Config Server secret release.

### 3. Remove infrastructure releases

Each infrastructure component is an independent Helm release in each matching
`*-infra` namespace. Check installed releases first:

```bash
helm list --namespace dev-infra
helm list --namespace test-infra
helm list --namespace uat-infra
```

Uninstall only releases that are installed and belong to this project:

```bash
helm uninstall mysql --namespace dev-infra
helm uninstall mongodb --namespace dev-infra
helm uninstall postgresql --namespace dev-infra
helm uninstall kafka --namespace dev-infra

helm uninstall mysql --namespace test-infra
helm uninstall mongodb --namespace test-infra
helm uninstall postgresql --namespace test-infra
helm uninstall kafka --namespace test-infra

helm uninstall mysql --namespace uat-infra
helm uninstall mongodb --namespace uat-infra
helm uninstall postgresql --namespace uat-infra
helm uninstall kafka --namespace uat-infra
```

Skip any release that is not installed. These charts manage PVCs as part of
their releases; Helm normally removes the PVC object on uninstall. Whether the
underlying storage and data are reclaimed depends on the cluster's storage
class and reclaim policy. Take and verify backups before uninstalling; inspect
PVCs and persistent volumes before any manual storage cleanup.

### 4. Remove namespaces only when empty and no longer needed

First confirm there are no remaining workloads, Secrets, ConfigMaps, or PVCs
and no resources from other projects in these namespaces. Deleting a namespace
deletes all resources in it:

```bash
kubectl get all,secret,configmap,pvc -n dev
kubectl get all,secret,configmap,pvc -n test
kubectl get all,secret,configmap,pvc -n uat
kubectl get all,secret,configmap,pvc -n dev-infra
kubectl get all,secret,configmap,pvc -n test-infra
kubectl get all,secret,configmap,pvc -n uat-infra
```

Only if these namespaces contain no unrelated resources, remove the namespace
definitions:

```bash
kubectl delete -f namespaces/
```

### 5. Remove local secret override files (optional)

Helm uninstall does not delete local files. Files matching `*.secret.yaml` are
ignored by Git and remain on disk. Remove only the specific local secret files
you no longer need, such as the MySQL, MongoDB, PostgreSQL, and
`gnax-config-server` secret overrides in `environments/<env>/`.

For team or CI usage, use an encrypted secret workflow such as SOPS with age or a secret manager such as Vault or External Secrets. Do not treat base64-encoded Kubernetes Secret manifests as encryption.
