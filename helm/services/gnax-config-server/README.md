# gnax-config-server

Helm chart for the GnaX Spring Cloud Config Server. Runs in the `dev` namespace; the Deployment and Service are named `config-server` (port 8888; `ClusterIP` by default, `NodePort` 30888 in `dev`).

## Prerequisites

- Rancher Desktop Kubernetes is running and the local context is selected.
- `config-server-secret` exists in `dev` (installed by
  [`helm/services/config-server`](../config-server/README.md)):

```bash
kubectl -n dev get secret config-server-secret
```

## Build the image

Run from the infrastructure repository root:

```bash
docker build -t gnax-config-server:0.0.1-SNAPSHOT /Users/rk/workspace/git/gnax/gnax-config-server
```

With the containerd runtime, use `nerdctl --namespace k8s.io build` instead.

## Deploy / upgrade

```bash
helm upgrade --install gnax-config-server ./helm/services/gnax-config-server \
  -n dev -f environments/dev/gnax-config-server.yaml
kubectl -n dev rollout status deployment/config-server
```

After rebuilding the image under the same tag, restart the pod:

```bash
kubectl -n dev rollout restart deployment/config-server
```

To use a new tag, update `image.tag` in `environments/dev/gnax-config-server.yaml` and re-run the upgrade.

## Validate

```bash
helm lint ./helm/services/gnax-config-server
helm template gnax-config-server ./helm/services/gnax-config-server \
  -n dev -f environments/dev/gnax-config-server.yaml
```
## Update 
To update the deployment with new values or a new image tag:

```bash
helm upgrade gnax-config-server ./helm/services/gnax-config-server \
  -n dev -f environments/dev/gnax-config-server.yaml
kubectl -n dev rollout status deployment/config-server
```
helm upgrade --install gnax-config-server ./helm/services/gnax-config-server -n dev -f environments/dev/gnax-config-server.yaml

## Access from your local machine

In `dev` the Service is a `NodePort` (set in `environments/dev/gnax-config-server.yaml`), so the config server is reachable from your browser and from applications running in your IDE:

```text
http://localhost:30888
```

Example: http://localhost:30888/actuator/health (the browser asks for the basic-auth username and password from `config-server-secret`).

Point IDE-run applications at it:

```properties
spring.config.import=optional:configserver:http://localhost:30888
spring.cloud.config.username=<username>
spring.cloud.config.password=<password>
```

Inside the cluster use `http://config-server.dev.svc.cluster.local:8888`.

If `localhost:30888` is not reachable, fall back to `kubectl -n dev port-forward svc/config-server 8888:8888` and use `http://localhost:8888`.

## Verify

```bash
kubectl -n dev get pods,svc -l app.kubernetes.io/name=config-server
curl -u <username>:<password> http://localhost:30888/actuator/health
```

In-cluster URL: `http://config-server.dev.svc.cluster.local:8888`

## Troubleshooting

```bash
kubectl -n dev logs deploy/config-server
kubectl -n dev describe pod -l app.kubernetes.io/name=config-server
```

## Uninstall

Uninstall the service release to remove the Config Server Deployment and
Service. Its `config-server-secret` is a separate release and is not removed by
this command:

```bash
helm uninstall gnax-config-server -n dev
```

When the secret is no longer needed, uninstall its release separately using
the instructions in [`helm/services/config-server/README.md`](../config-server/README.md).
