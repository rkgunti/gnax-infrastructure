# Config Server secret

This chart installs the Kubernetes Secret `config-server-secret` as an
independent Helm release named `config-server` in the application namespace
(currently `dev`). It does not deploy a Pod, Deployment, or Service. The
GnaX Config Server Deployment consumes the Secret and needs it at startup.

## Values and secret file

The secret contains `CONFIG_SERVER_USERNAME`, `CONFIG_SERVER_PASSWORD`, and
`CONFIG_GIT_URI`. Create the ignored local override file from the example and
replace every placeholder:

```bash
cp environments/dev/config-server.secret.yaml.example \
  environments/dev/config-server.secret.yaml
```

The tracked environment file enables the chart:

```text
environments/dev/config-server.yaml
```

Never commit the local `*.secret.yaml` override. Kubernetes Secret data is
base64-encoded and is not automatically encrypted by this chart.

## Validate

```bash
helm lint ./helm/services/config-server \
  --set configServer.username=lint-only \
  --set configServer.password=lint-only \
  --set configServer.gitUri=https://example.invalid/config.git
helm template config-server ./helm/services/config-server \
  --namespace dev \
  --values environments/dev/config-server.yaml \
  --values environments/dev/config-server.secret.yaml
```

## Install or update

```bash
helm upgrade --install config-server ./helm/services/config-server \
  --namespace dev \
  --create-namespace \
  --values environments/dev/config-server.yaml \
  --values environments/dev/config-server.secret.yaml
kubectl get secret config-server-secret --namespace dev
```

Use the matching environment file and namespace if installing for another
environment. Install this Secret before deploying or restarting the
GnaX Config Server. See its [service README](../gnax-config-server/README.md).

## Uninstall

First uninstall the GnaX Config Server application release as described in
its [service README](../gnax-config-server/README.md). Then remove this
independent release:

```bash
helm uninstall config-server --namespace dev
```

This removes the Helm-managed Kubernetes Secret, not the local
`environments/dev/config-server.secret.yaml` file. Remove that ignored local
file separately if it is no longer needed. Do not uninstall this release while
the Config Server still needs the Secret to start.
