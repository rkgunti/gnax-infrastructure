# GnaX Identity Service

Helm chart for the GnaX identity service. One release manages the service's
Secret, Deployment, and Service. Its shared configuration and tracked secret
template are in `values/services/gnax-identity-service/values.yaml` and
`values/services/gnax-identity-service/secret.yaml.example`. Create the
ignored secret values file alongside that template before installing.

The current local install uses namespace `dev`, release and resource name
`gnax-identity-service`, container port `8080`, and NodePort `30081`.

## Prepare credentials and keys

Create the ignored secret values file and fill in the Config Server
username and password:

```bash
cp values/services/gnax-identity-service/secret.yaml.example \
  values/services/gnax-identity-service/secret.yaml
```

The chart sets `SPRING_APPLICATION_NAME` to `gnax-identity-service` and
provides `CONFIG_SERVER_URL` from the shared values file. Use
`${CONFIG_SERVER_URL:configserver:http://localhost:30888}` for
`spring.config.import` in the application. The default URL is the Config
Server's in-cluster address for the current `dev` namespace.

## Build

From the infrastructure repository root, build the service image (adjust the
sibling source path if needed):

```bash
docker build --build-arg APP_PORT=8080 \
  -t gnax-identity-service:0.0.1-SNAPSHOT ../gnax-identity-service
```

For Rancher Desktop's containerd runtime, use
`nerdctl --namespace k8s.io build` instead.

## Validate and install

Lint, render, or install using the same values files:

```bash
helm lint ./helm/services/gnax-identity-service \
  --values values/services/gnax-identity-service/values.yaml \
  --values values/services/gnax-identity-service/secret.yaml
helm template gnax-identity-service ./helm/services/gnax-identity-service \
  --namespace dev \
  --values values/services/gnax-identity-service/values.yaml \
  --values values/services/gnax-identity-service/secret.yaml
helm upgrade --install gnax-identity-service ./helm/services/gnax-identity-service \
  --namespace dev --create-namespace \
  --values values/services/gnax-identity-service/values.yaml \
  --values values/services/gnax-identity-service/secret.yaml
kubectl -n dev rollout status deployment/gnax-identity-service
kubectl -n dev get pods,svc -l app.kubernetes.io/name=gnax-identity-service
```

## Access and operations

Host-side tools can reach the service at `http://localhost:30081`:

```bash
curl http://localhost:30081/actuator/health/readiness
```

In-cluster clients use
`http://gnax-identity-service.dev.svc.cluster.local:8080`.

After changing credentials, keys, or chart configuration, run the Helm upgrade
again and restart the Deployment so it reloads the updated Secret:

```bash
kubectl -n dev rollout restart deployment/gnax-identity-service
kubectl -n dev rollout status deployment/gnax-identity-service
```

To use a new image tag, update `image.tag` in
`values/services/gnax-identity-service/values.yaml` and upgrade the release.

## Troubleshooting and uninstall

```bash
kubectl -n dev logs deploy/gnax-identity-service
kubectl -n dev describe pod -l app.kubernetes.io/name=gnax-identity-service
kubectl -n dev get pods,svc,secret -l app.kubernetes.io/name=gnax-identity-service
helm uninstall gnax-identity-service --namespace dev
```

Uninstalling removes the release-managed Secret, Deployment, and Service. The local
secret values file is not removed automatically. Do not delete the
namespace to remove this service if other releases use it.
