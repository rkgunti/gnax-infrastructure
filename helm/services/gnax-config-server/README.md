# GnaX Config Server

Helm chart for the GnaX Spring Cloud Config Server. It deploys the Config
Server's Kubernetes Secret, Deployment, and Service together as one release.
In `dev`, the Helm release is `gnax-config-server`; the Deployment, Service,
and Secret are named `config-server`, `config-server`, and
`config-server-secret`. The application listens on port `8888`; the dev
Service exposes NodePort `30888`.

## Prerequisites

- Rancher Desktop Kubernetes is running and the local context is selected.
- `kubectl`, Helm 3 or later, and Docker (or `nerdctl` for containerd) are
  installed.
- The ignored values file
  `environments/dev/gnax-config-server.secret.yaml` exists and contains local
  credentials and a Git URI:

```bash
cp environments/dev/gnax-config-server.secret.yaml.example \
  environments/dev/gnax-config-server.secret.yaml
```

Replace all `CHANGE_ME_*` values. Never commit the local secret file.

## Build the image

Run from the infrastructure repository root, adjusting the sibling source
directory if it is located elsewhere:

```bash
docker build -t gnax-config-server:0.0.1-SNAPSHOT ../gnax-config-server
```

With the Rancher Desktop containerd runtime, use
`nerdctl --namespace k8s.io build` instead.

## Validate

Lint and render using the same environment and secret values that will be used
for deployment:

```bash
helm lint ./helm/services/gnax-config-server \
  --values environments/dev/gnax-config-server.yaml \
  --values environments/dev/gnax-config-server.secret.yaml
helm template gnax-config-server ./helm/services/gnax-config-server \
  --namespace dev \
  --values environments/dev/gnax-config-server.yaml \
  --values environments/dev/gnax-config-server.secret.yaml
```

## Install or update

One release manages the Secret, Deployment, and Service. The secret values are
applied by Helm from the ignored values file:

```bash
helm upgrade --install gnax-config-server ./helm/services/gnax-config-server \
  --namespace dev \
  --create-namespace \
  --values environments/dev/gnax-config-server.yaml \
  --values environments/dev/gnax-config-server.secret.yaml
kubectl -n dev rollout status deployment/config-server
kubectl -n dev get pods,svc -l app.kubernetes.io/name=config-server
kubectl -n dev get secret config-server-secret
```

### Migrate from the former separate secret release

If this cluster still has a Helm release named `config-server` in `dev`, that
release owns `config-server-secret`. The Secret must not be owned by two Helm
releases. Keep the local secret values file ready, then run these commands
back-to-back. The currently running Config Server Pod keeps its already-loaded
environment, but avoid restarting or rescheduling it during the short interval
between commands:

```bash
helm uninstall config-server --namespace dev
helm upgrade --install gnax-config-server ./helm/services/gnax-config-server \
  --namespace dev \
  --values environments/dev/gnax-config-server.yaml \
  --values environments/dev/gnax-config-server.secret.yaml
kubectl -n dev rollout restart deployment/config-server
kubectl -n dev rollout status deployment/config-server
```

The first command removes only the old Secret release; the second makes the
new service release own the Secret together with the application. Do not run
the migration command if `helm list --namespace dev` shows no old
`config-server` release.

To change credentials or the Git URI, edit the local secret values file and
rerun the Helm upgrade command, then restart the Deployment so it reloads the
updated Secret:

```bash
kubectl -n dev rollout restart deployment/config-server
kubectl -n dev rollout status deployment/config-server
```

After rebuilding the image with the same tag, also restart the Deployment. To
use a new image tag, update `image.tag` in
`environments/dev/gnax-config-server.yaml` and run the Helm upgrade command.

## Access

In `dev`, host-side tools can reach the server at
`http://localhost:30888`. The health endpoint requires the basic-auth
credentials from `gnax-config-server.secret.yaml`:

```bash
curl -u <username>:<password> http://localhost:30888/actuator/health
```

Inside Kubernetes, use
`http://config-server.dev.svc.cluster.local:8888`. If the NodePort is not
reachable, forward the Service port:

```bash
kubectl -n dev port-forward svc/config-server 8888:8888
```

## Troubleshooting

```bash
kubectl -n dev logs deploy/config-server
kubectl -n dev describe pod -l app.kubernetes.io/name=config-server
kubectl -n dev get pods,svc -l app.kubernetes.io/name=config-server
kubectl -n dev get secret config-server-secret
```

## Uninstall

Check the release and its resources before uninstalling:

```bash
helm list --namespace dev
kubectl -n dev get pods,svc,secret,pvc
```

Uninstall the single release to remove its Deployment, Service, and
`config-server-secret` together:

```bash
helm uninstall gnax-config-server --namespace dev
```

The release does not manage a PVC. The ignored local values file
`environments/dev/gnax-config-server.secret.yaml` remains on disk; remove it
separately if it is no longer needed. Do not delete the `dev` namespace just
to remove this service because it can contain other application releases.
