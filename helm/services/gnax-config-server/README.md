# GnaX Config Server

Helm chart for the GnaX Spring Cloud Config Server. One release manages its
Secret, Deployment, and Service. The current shared values files are
`values/services/gnax-config-server/values.yaml` and
`values/services/gnax-config-server/secret.yaml`; the tracked secret template
is `values/services/gnax-config-server/secret.yaml.example`.

The local install uses namespace `dev`, release `gnax-config-server`, resource
name `config-server`, port `8888`, and NodePort `30888`.

## Prepare secrets

Create the ignored secret values file and replace all placeholders:

```bash
cp values/services/gnax-config-server/secret.yaml.example \
  values/services/gnax-config-server/secret.yaml
```

Never commit the local secret file.

## Build

From the infrastructure repository root, build the service image (adjust the
sibling source path if needed):

```bash
docker build --build-arg APP_PORT=8080 -t gnax-config-server:0.0.1-SNAPSHOT ../gnax-config-server
```

For Rancher Desktop's containerd runtime, use
`nerdctl --namespace k8s.io build` instead.

## Validate and install

```bash
helm lint ./helm/services/gnax-config-server \
  --values values/services/gnax-config-server/values.yaml \
  --values values/services/gnax-config-server/secret.yaml
helm template gnax-config-server ./helm/services/gnax-config-server \
  --namespace dev \
  --values values/services/gnax-config-server/values.yaml \
  --values values/services/gnax-config-server/secret.yaml
helm upgrade --install gnax-config-server ./helm/services/gnax-config-server \
  --namespace dev --create-namespace \
  --values values/services/gnax-config-server/values.yaml \
  --values values/services/gnax-config-server/secret.yaml
kubectl -n dev rollout status deployment/gnax-config-server
kubectl -n dev get pods,svc -l app.kubernetes.io/name=gnax-config-server
```

If the cluster still has an older, separate Helm release named `config-server`
in `dev` managing `gnax-config-server-secret`, uninstall that release before
installing this chart. Do not run this migration if that old release is absent:

```bash
helm uninstall gnax-config-server --namespace dev
helm upgrade --install gnax-config-server ./helm/services/gnax-config-server \
  --namespace dev \
  --values values/services/gnax-config-server/values.yaml \
  --values values/services/gnax-config-server/secret.yaml
```

## Access and operations

Host-side tools can reach the server at `http://localhost:30888`. The health
endpoint uses the basic-auth credentials in the local secret values file:

```bash
curl -u <username>:<password> http://localhost:30888/actuator/health
```

In-cluster clients use
`http://config-server.dev.svc.cluster.local:8888`.

When changing credentials or the Git URI, upgrade the release and restart the
Deployment so the process reloads the Secret:

```bash
helm upgrade --install gnax-config-server ./helm/services/gnax-config-server \
  --namespace dev \
  --values values/services/gnax-config-server/values.yaml \
  --values values/services/gnax-config-server/secret.yaml
kubectl -n dev rollout restart deployment/config-server
kubectl -n dev rollout status deployment/config-server
```

To use a new image tag, update `image.tag` in
`values/services/gnax-config-server/values.yaml` and upgrade the release.

## Troubleshooting and uninstall

```bash
kubectl -n dev logs deploy/gnax-config-server
kubectl -n dev describe pod -l app.kubernetes.io/name=gnax-config-server
kubectl -n dev get pods,svc,secret -l app.kubernetes.io/name=gnax-config-server
helm uninstall gnax-config-server --namespace dev
```

Uninstalling removes the release-managed Secret, Deployment, and Service. The
ignored local secret values file remains on disk; remove it separately if no
longer needed. Do not delete the namespace to remove this service if other
releases use it.
