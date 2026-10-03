# Kafka

This chart deploys a single-node Kafka broker in KRaft mode as an independent
Helm release in a local infrastructure namespace. It is intended for
Rancher Desktop development and testing, not production use. The local broker
uses plaintext listeners and has no authentication.

## Per-environment settings

| Environment | Namespace | Host NodePort |
| --- | --- | --- |
| dev | `dev-infra` | `30094` |
| test | `test-infra` | `30094` |
| uat | `uat-infra` | `30094` |

In-cluster clients use port `9092`; the chart also configures the host listener
on port `9094`. The Service and Deployment are named `kafka`; the chart creates
an `8Gi` PVC named `kafka-pvc`.

## Install or update

Ensure the environment namespaces have been created with `kubectl apply -f namespaces/`.
For dev:

```bash
helm upgrade --install kafka ./helm/infra/kafka \
  --namespace dev-infra \
  --values environments/dev/kafka.yaml
```

For test or uat, use the matching environment name in the namespace and values
file. Each environment is a separate release in its own namespace.

Validate the chart and check the workload:

```bash
helm lint ./helm/infra/kafka
kubectl get pods,svc -n dev-infra -l app=kafka
kubectl get pvc kafka-pvc -n dev-infra
kubectl logs deployment/kafka -n dev-infra
```

## Connect

In-cluster clients use `kafka.<env>-infra.svc.cluster.local:9092`, for example
`kafka.dev-infra.svc.cluster.local:9092`. Host-side clients use
`localhost:30094`. The advertised host listener uses `localhost`, so external
clients should run on the same host as Rancher Desktop.

## Uninstall

Check the release and storage first:

```bash
helm list --namespace dev-infra
kubectl get pvc -n dev-infra
```

Back up any needed topics and data before uninstalling. The release manages
the `kafka-pvc`; Helm normally removes that PVC along with its Deployment and
Service. Underlying data reclamation depends on the storage class and its
reclaim policy.

```bash
helm uninstall kafka --namespace dev-infra
```

For test or uat, use the corresponding `test-infra` or `uat-infra` namespace.
