# GnaX Identity Service

Helm chart for the GnaX identity service. It deploys the service's Kubernetes
Secret, Deployment, and Service together as one release. In `dev`, the Helm
release is `gnax-identity-service`; the Deployment, Service, and Secret are
named `gnax-identity-service`, `gnax-identity-service`, and
`gnax-identity-service-secret`. The application listens on port `8080`; the
dev Service exposes NodePort `30081`.

## Prerequisites

- Rancher Desktop Kubernetes is running and the local context is selected.
- `kubectl`, Helm 3 or later, and Docker (or `nerdctl` for containerd) are
  installed.
- The ignored values file
  `environments/dev/gnax-identity-service.secret.yaml` exists and contains the
  database and Config Server credentials (`DB_USER`, `DB_USER_PASSWORD`,
  `CONFIG_SERVER_USERNAME`, `CONFIG_SERVER_PASSWORD` in the Secret). Keep the
  JWT PEM files separately under the git-ignored `secrets/dev/` directory.

```bash
cp environments/dev/gnax-identity-service.secret.yaml.example \
  environments/dev/gnax-identity-service.secret.yaml
```

Replace all `CHANGE_ME_*` values. Never commit the local secret file.

The environment-specific values file supplies `CONFIG_SERVER_URL` to the
container (for example, `configserver:http://config-server.dev.svc.cluster.local:8888`).
The application can use `${CONFIG_SERVER_URL:configserver:http://localhost:30888}`
for `spring.config.import`; the chart also sets `SPRING_APPLICATION_NAME` to
`gnax-identity-service`. Update `identity.configServerUrl` in the matching
`environments/<env>/gnax-identity-service.yaml` file when the Config Server
address changes.

Generate a JWT key pair if you do not have one:

```bash
openssl genpkey -algorithm RSA -out private_key.pem -pkeyopt rsa_keygen_bits:2048
openssl rsa -pubout -in private_key.pem -out public_key.pem
```

Do not put PEM contents or paths in the secret values file. Pass PEM file paths
to Helm with `--set-file`; Helm reads the files and stores their contents in the
Kubernetes Secret as `JWT_PRIVATE_KEY` and `JWT_PUBLIC_KEY`.

## Build the image

Run from the infrastructure repository root, adjusting the sibling source
directory if it is located elsewhere:

```bash
docker build --build-arg APP_PORT=8080 -t gnax-identity-service:0.0.1-SNAPSHOT ../gnax-identity-service
```

The Dockerfile is generic (copies `target/*.jar`), so the same file and command
pattern work for every service; change only the image name, source directory,
and `APP_PORT`.

With the Rancher Desktop containerd runtime, use
`nerdctl --namespace k8s.io build` instead.

## Validate

Lint and render using the same environment and secret values that will be used
for deployment:

```bash
helm lint ./helm/services/gnax-identity-service \
  --values environments/dev/gnax-identity-service.yaml \
  --values environments/dev/gnax-identity-service.secret.yaml \
  --set-file identity.jwtPrivateKey=secrets/dev/private_key.pem \
  --set-file identity.jwtPublicKey=secrets/dev/public_key.pem
helm template gnax-identity-service ./helm/services/gnax-identity-service \
  --namespace dev \
  --values environments/dev/gnax-identity-service.yaml \
  --values environments/dev/gnax-identity-service.secret.yaml \
  --set-file identity.jwtPrivateKey=secrets/dev/private_key.pem \
  --set-file identity.jwtPublicKey=secrets/dev/public_key.pem
```

## Install or update

One release manages the Secret, Deployment, and Service. The secret values are
applied by Helm from the ignored values file:

```bash
helm upgrade --install gnax-identity-service ./helm/services/gnax-identity-service \
  --namespace dev \
  --create-namespace \
  --values environments/dev/gnax-identity-service.yaml \
  --values environments/dev/gnax-identity-service.secret.yaml \
  --set-file identity.jwtPrivateKey=secrets/dev/private_key.pem \
  --set-file identity.jwtPublicKey=secrets/dev/public_key.pem
kubectl -n dev rollout status deployment/gnax-identity-service
kubectl -n dev get pods,svc -l app.kubernetes.io/name=gnax-identity-service
kubectl -n dev get secret gnax-identity-service-secret
```

To supply the JWT keys from PEM files instead of the values file, add:

```bash
  --set-file identity.jwtPrivateKey=secrets/dev/private_key.pem \
  --set-file identity.jwtPublicKey=secrets/dev/public_key.pem
```

To change credentials or keys, edit the local secret values file and rerun the
Helm upgrade command, then restart the Deployment so it reloads the updated
Secret:

```bash
kubectl -n dev rollout restart deployment/gnax-identity-service
kubectl -n dev rollout status deployment/gnax-identity-service
```

After rebuilding the image with the same tag, also restart the Deployment. To
use a new image tag, update `image.tag` in
`environments/dev/gnax-identity-service.yaml` and run the Helm upgrade command.

## Access

In `dev`, host-side tools can reach the service at
`http://localhost:30081`:

```bash
curl http://localhost:30081/actuator/health/readiness
```

Inside Kubernetes, use
`http://gnax-identity-service.dev.svc.cluster.local:8080`. If the NodePort is
not reachable, forward the Service port:

```bash
kubectl -n dev port-forward svc/gnax-identity-service 8080:8080
```

## Troubleshooting

```bash
kubectl -n dev logs deploy/gnax-identity-service
kubectl -n dev describe pod -l app.kubernetes.io/name=gnax-identity-service
kubectl -n dev get pods,svc -l app.kubernetes.io/name=gnax-identity-service
kubectl -n dev get secret gnax-identity-service-secret
```

## Uninstall

Check the release and its resources before uninstalling:

```bash
helm list --namespace dev
kubectl -n dev get pods,svc,secret,pvc
```

Uninstall the single release to remove its Deployment, Service, and
`gnax-identity-service-secret` together:

```bash
helm uninstall gnax-identity-service --namespace dev
```

The release does not manage a PVC. The ignored local values file
`environments/dev/gnax-identity-service.secret.yaml` remains on disk; remove it
separately if it is no longer needed. Do not delete the `dev` namespace just
to remove this service because it can contain other application releases.
