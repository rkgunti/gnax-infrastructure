# gnax-infrastructure

Local Kubernetes infrastructure and Helm charts for GnaX services and
dependencies. Current configuration uses one shared values set rather than
parallel `dev`, `test`, and `uat` values directories. Values are grouped by
component type:

- `values/infra/` contains infrastructure configuration and secret examples.
- `values/services/` contains application service configuration and secret
  examples.
- Local files ending in `.secret.yaml` are ignored by Git; keep credentials
  there, never in chart defaults or tracked values.

The checked-in namespace manifests still provide the current local targets:
application services in `dev` and infrastructure in `dev-infra`. Environment
branches can later carry branch-specific values without duplicating the
configuration layout.

## Repository structure

```text
helm/
  infra/
    kafka/
    mongodb/
    mysql/
    postgresql/
  services/
    gnax-config-server/
    gnax-identity-service/
values/
  infra/
    kafka.yaml
    mongodb.yaml
    mongodb.secret.yaml.example
    mysql.yaml
    mysql.secret.yaml.example
    postgresql.yaml
    postgresql.secret.yaml.example
  services/
    gnax-config-server.yaml
    gnax-config-server.secret.yaml.example
    gnax-identity-service.yaml
    gnax-identity-service.secret.yaml.example
namespaces/
```

Use generic service names such as `orders`, `catalog`, or `notifications`.
Avoid architectural names such as `frontend` or `backend`.

## Prerequisites and namespaces

Install Rancher Desktop with Kubernetes enabled, plus `kubectl`, Helm 3 or
later, and Docker (or `nerdctl` for containerd). Confirm the current context
points to the local cluster:

```bash
kubectl config current-context
kubectl get nodes
helm version
```

Create the current local namespaces:

```bash
kubectl apply -f namespaces/dev.yaml
kubectl apply -f namespaces/dev-infra.yaml
```

Each database, broker, and service remains an independent Helm release.

## Prepare local secrets

Create ignored secret override files from the tracked examples:

```bash
cp values/infra/mysql.secret.yaml.example values/infra/mysql.secret.yaml
cp values/infra/mongodb.secret.yaml.example values/infra/mongodb.secret.yaml
cp values/infra/postgresql.secret.yaml.example values/infra/postgresql.secret.yaml
cp values/services/gnax-config-server/secret.yaml.example values/services/gnax-config-server/secret.yaml
cp values/services/gnax-identity-service/secret.yaml.example values/services/gnax-identity-service/secret.yaml
```

Replace every `CHANGE_ME_*` value with local credentials. Never commit the
resulting `.secret.yaml` files.

## Validate charts

Lint the charts with the local secret overrides:

```bash
helm lint ./helm/infra/mysql \
  --values values/infra/mysql.yaml \
  --values values/infra/mysql.secret.yaml
helm lint ./helm/infra/mongodb \
  --values values/infra/mongodb.yaml \
  --values values/infra/mongodb.secret.yaml
helm lint ./helm/infra/postgresql \
  --values values/infra/postgresql.yaml \
  --values values/infra/postgresql.secret.yaml
helm lint ./helm/infra/kafka --values values/infra/kafka.yaml
helm lint ./helm/services/gnax-config-server \
  --values values/services/gnax-config-server/values.yaml \
  --values values/services/gnax-config-server/secret.yaml
helm lint ./helm/services/gnax-identity-service \
  --values values/services/gnax-identity-service/values.yaml \
  --values values/services/gnax-identity-service/secret.yaml
```

To render a chart before installing, use `helm template` with the same values.
For example:

```bash
helm template mysql ./helm/infra/mysql \
  --namespace dev-infra \
  --values values/infra/mysql.yaml \
  --values values/infra/mysql.secret.yaml
```

## Install or update

Install infrastructure components independently into `dev-infra`:

```bash
helm upgrade --install mysql ./helm/infra/mysql \
  --namespace dev-infra --create-namespace \
  --values values/infra/mysql.yaml \
  --values values/infra/mysql.secret.yaml
helm upgrade --install mongodb ./helm/infra/mongodb \
  --namespace dev-infra \
  --values values/infra/mongodb.yaml \
  --values values/infra/mongodb.secret.yaml
helm upgrade --install postgresql ./helm/infra/postgresql \
  --namespace dev-infra \
  --values values/infra/postgresql.yaml \
  --values values/infra/postgresql.secret.yaml
helm upgrade --install kafka ./helm/infra/kafka \
  --namespace dev-infra \
  --values values/infra/kafka.yaml
```

Install the Config Server and identity service independently into `dev`:

```bash
helm upgrade --install gnax-config-server ./helm/services/gnax-config-server \
  --namespace dev --create-namespace \
  --values values/services/gnax-config-server/values.yaml \
  --values values/services/gnax-config-server/secret.yaml
helm upgrade --install gnax-identity-service ./helm/services/gnax-identity-service \
  --namespace dev \
  --values values/services/gnax-identity-service/values.yaml \
  --values values/services/gnax-identity-service/secret.yaml
```

Check rollout and resources:

```bash
kubectl -n dev get pods,svc
kubectl -n dev-infra get pods,svc,pvc
```

The Config Server and identity service chart READMEs contain build, access,
troubleshooting, and uninstall instructions:

- [GnaX Config Server](./helm/services/gnax-config-server/README.md)
- [GnaX Identity Service](./helm/services/gnax-identity-service/README.md)
- [MySQL](./helm/infra/mysql/README.md)
- [MongoDB](./helm/infra/mongodb/README.md)
- [PostgreSQL](./helm/infra/postgresql/README.md)
- [Kafka](./helm/infra/kafka/README.md)

## Add another service or infrastructure component

Keep chart defaults and reusable templates under `helm/`. Add the active
configuration under the matching values category, for example:

```text
values/services/<service-name>/values.yaml
values/services/<service-name>/secret.yaml.example
values/infra/<component>.yaml
values/infra/<component>.secret.yaml.example
```

Create a local ignored `.secret.yaml` file from its example when secrets are
required, and pass the config and secret files separately with `--values`.
Keep deployments as independent Helm releases. For a new application, use a
`ClusterIP` Service by default and add `NodePort` only when host access is
needed.
