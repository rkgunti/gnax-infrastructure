# Kafka

This chart deploys a single-node Kafka broker in KRaft mode as an independent
Helm release in a local infrastructure namespace. It is intended for
Rancher Desktop development and testing, not production use. The local broker
uses plaintext listeners and has no authentication.

## Current local settings

The release uses namespace `dev-infra` and host NodePort `30094`. Its settings
are in `values/infra/kafka.yaml`.

In-cluster clients use port `9092`; the chart also configures the host listener
on port `9094`. The Service and Deployment are named `kafka`; the chart creates
an `8Gi` PVC named `kafka-pvc`.

## Install or update

Ensure namespace `dev-infra` exists, then install or update the release:

```bash
helm upgrade --install kafka ./helm/infra/kafka \
  --namespace dev-infra \
  --values values/infra/kafka.yaml
```

Validate the chart and check the workload:

```bash
helm lint ./helm/infra/kafka
kubectl get pods,svc -n dev-infra -l app=kafka
kubectl get pvc kafka-pvc -n dev-infra
kubectl logs deployment/kafka -n dev-infra
```

## Connect

In-cluster clients use `kafka.dev-infra.svc.cluster.local:9092`. Host-side
clients use `localhost:30094`. The advertised host listener uses `localhost`, so external
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
