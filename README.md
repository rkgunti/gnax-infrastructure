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
		kafka/
	services/
		<service-name>/

environments/
	dev/
		mysql.yaml
		mysql.secret.yaml.example
		mongodb.yaml
		mongodb.secret.yaml.example
		kafka.yaml
		<service-name>.yaml
	test/
		mysql.yaml
		mysql.secret.yaml.example
		mongodb.yaml
		mongodb.secret.yaml.example
		kafka.yaml
		<service-name>.yaml
	uat/
		mysql.yaml
		mysql.secret.yaml.example
		mongodb.yaml
		mongodb.secret.yaml.example
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
cp environments/test/mysql.secret.yaml.example environments/test/mysql.secret.yaml
cp environments/test/mongodb.secret.yaml.example environments/test/mongodb.secret.yaml
cp environments/uat/mysql.secret.yaml.example environments/uat/mysql.secret.yaml
cp environments/uat/mongodb.secret.yaml.example environments/uat/mongodb.secret.yaml
for e in dev test uat; do cp environments/$e/config-server.secret.yaml.example environments/$e/config-server.secret.yaml; done
```

Open each new `mysql.secret.yaml` and `mongodb.secret.yaml` file and replace every `CHANGE_ME_*` value with a local password. Never commit these files. They are ignored by `.gitignore`.

## Step 3: Validate the Infrastructure Charts

Lint each independent chart:

```bash
helm lint ./helm/infra/mysql
helm lint ./helm/infra/mongodb
helm lint ./helm/infra/kafka
helm lint ./helm/infra/config-server
```

Render each chart independently. MySQL and MongoDB require their matching secret file:

```bash
helm template mysql ./helm/infra/mysql \
	--namespace dev-infra \
	--values ./environments/dev/mysql.yaml \
	--values ./environments/dev/mysql.secret.yaml

helm template mongodb ./helm/infra/mongodb \
	--namespace dev-infra \
	--values ./environments/dev/mongodb.yaml \
	--values ./environments/dev/mongodb.secret.yaml

helm template kafka ./helm/infra/kafka \
	--namespace dev-infra \
	--values ./environments/dev/kafka.yaml

helm template config-server ./helm/infra/config-server \
	--namespace dev \
	--values ./environments/dev/config-server.yaml \
	--values ./environments/dev/config-server.secret.yaml
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

Deploy or update the Config Server secret (`config-server-secret`, keys `CONFIG_SERVER_USERNAME`, `CONFIG_SERVER_PASSWORD`, `CONFIG_GIT_URI`). Reference it from the config server deployment with `envFrom.secretRef.name: config-server-secret`; the Spring `${...}` placeholders then resolve from these env vars:

```bash
helm upgrade --install config-server ./helm/infra/config-server \
	--namespace dev \
	--create-namespace \
	--values ./environments/dev/config-server.yaml \
	--values ./environments/dev/config-server.secret.yaml
```

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

The config server runs in the `dev` namespace as the `gnax-config-server` Helm release (Kubernetes Deployment and Service name: `config-server`).

1. Create the secret file and set the credentials and Git URI (the file is git-ignored):

```bash
cp environments/dev/config-server.secret.yaml.example environments/dev/config-server.secret.yaml
```

2. Install the `config-server-secret` into `dev` and confirm it exists:

```bash
helm upgrade --install config-server ./helm/infra/config-server \
	--namespace dev \
	--create-namespace \
	--values ./environments/dev/config-server.yaml \
	--values ./environments/dev/config-server.secret.yaml
kubectl -n dev get secret config-server-secret
```

3. Build the image so the local cluster can use it (Rancher Desktop with dockerd; for containerd use `nerdctl --namespace k8s.io build`):

```bash
docker build -t gnax-config-server:0.0.1-SNAPSHOT /Users/rk/workspace/git/gnax/gnax-config-server
```

4. Validate the chart:

```bash
helm lint ./helm/services/gnax-config-server
helm template gnax-config-server ./helm/services/gnax-config-server \
	--namespace dev \
	--values ./environments/dev/gnax-config-server.yaml
```

5. Deploy the application and wait for it:

```bash
helm upgrade --install gnax-config-server ./helm/services/gnax-config-server \
	--namespace dev \
	--values ./environments/dev/gnax-config-server.yaml
kubectl -n dev rollout status deployment/config-server
```

6. Verify (use the credentials from the secret file):

```bash
kubectl -n dev get pods,svc -l app.kubernetes.io/name=config-server
kubectl -n dev port-forward svc/config-server 8888:8888
curl -u <username>:<password> http://localhost:8888/actuator/health
```

In-cluster URL for other services: `http://config-server.dev.svc.cluster.local:8888` (basic auth from the secret).

To change config only, re-run step 5; after code changes, rebuild the image (step 3, optionally with a new tag in `environments/dev/gnax-config-server.yaml`) and re-run step 5. If the pod fails to start, check `kubectl -n dev logs deploy/config-server`.

## Connection Details

Services inside the cluster use Kubernetes DNS:

```text
mongodb.dev-infra.svc.cluster.local:27017
mysql.dev-infra.svc.cluster.local:3306
kafka.dev-infra.svc.cluster.local:9092
```

Replace `dev-infra` with `test-infra` or `uat-infra` for the other environments.

Host-side tools use these NodePorts:

```text
Environment  MongoDB    MySQL       Kafka
dev          30017      30306      30094
test         30018      30307      30094
uat          30019      30308      30094
```

Kafka clients inside the cluster use `kafka.<env>-infra.svc.cluster.local:9092`. Host-side tools use `127.0.0.1:30094`.

Example local MongoDB connection:

```text
mongodb://<user>:<password>@127.0.0.1:30017/<database>
```

## Troubleshooting

```bash
kubectl get pods -A
kubectl describe pod <pod-name> -n <namespace>
kubectl logs deployment/mongodb -n dev-infra
kubectl logs deployment/mysql -n dev-infra
kubectl logs deployment/kafka -n dev-infra
kubectl get endpoints -n dev-infra
helm get manifest mysql -n dev-infra
helm get manifest mongodb -n dev-infra
helm get manifest kafka -n dev-infra
```

Database PVCs are namespace-scoped. Do not move a database to another namespace without a backup and restore plan. For disposable local data, uninstall the old release and install the new one only when data loss is acceptable.

## Complete Cleanup

See [UNINSTALL.md](UNINSTALL.md) for the step-by-step procedure to remove service releases, database releases, PVCs, namespaces, and local secret files.

For team or CI usage, use an encrypted secret workflow such as SOPS with age or a secret manager such as Vault or External Secrets. Do not treat base64-encoded Kubernetes Secret manifests as encryption.
