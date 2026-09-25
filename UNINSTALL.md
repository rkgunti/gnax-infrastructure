# Uninstall and Cleanup

This document removes the infrastructure resources managed by this repository from the local Rancher Desktop Kubernetes cluster.

The commands below can delete deployments, Services, Secrets, ConfigMaps, PVCs, namespaces, and database data. Review each inventory command before running a destructive step.

## 1. Confirm the Kubernetes Context

```bash
kubectl config current-context
kubectl get nodes
```

Stop if the context is not the intended local Rancher Desktop cluster.

## 2. Inventory Current Resources

List Helm releases and resources before deleting anything:

```bash
helm list -A
kubectl get namespaces
kubectl get pods,svc,pvc -A
```

The repository namespaces are:

```text
dev
test
uat
dev-infra
test-infra
uat-infra
```

Do not delete a namespace if it contains resources managed outside this repository.

## 3. Remove Application Service Releases

Application services are installed into `dev`, `test`, or `uat`. List releases in each namespace:

```bash
helm list -n dev
helm list -n test
helm list -n uat
```

Uninstall only the service releases that belong to this infrastructure:

```bash
helm uninstall <service-name> --namespace dev
helm uninstall <service-name> --namespace test
helm uninstall <service-name> --namespace uat
```

Repeat for every service release in each environment. Do not uninstall unrelated releases such as cluster ingress or Rancher-managed components.

If services were installed with a different release name, use the name shown by `helm list`.

## 4. Remove Database Releases

MySQL, MongoDB, and Kafka are independent Helm releases. Check first:

```bash
helm list -n dev-infra
helm list -n test-infra
helm list -n uat-infra
```

Uninstall the releases that are installed in each database namespace:

```bash
helm uninstall mysql --namespace dev-infra
helm uninstall mongodb --namespace dev-infra
helm uninstall kafka --namespace dev-infra

helm uninstall mysql --namespace test-infra
helm uninstall mongodb --namespace test-infra
helm uninstall kafka --namespace test-infra

helm uninstall mysql --namespace uat-infra
helm uninstall mongodb --namespace uat-infra
helm uninstall kafka --namespace uat-infra
```

Only run commands for releases that exist. A missing release is harmless; check `helm list -n <namespace>` first.

## 5. Decide What to Do With Database and Kafka PVCs

Inspect remaining PVCs, including Kafka data:

```bash
kubectl get pvc -n dev-infra
kubectl get pvc -n test-infra
kubectl get pvc -n uat-infra
```

If database data must be preserved, stop here and create a backup before deleting PVCs.

For a complete disposable local reset, delete the database and Kafka PVCs:

```bash
kubectl delete pvc --all -n dev-infra
kubectl delete pvc --all -n test-infra
kubectl delete pvc --all -n uat-infra
```

Deleting PVCs permanently removes the local database data when the underlying storage is reclaimed.

## 6. Remove Repository Namespaces

After all service and database releases are removed, delete the six namespaces:

```bash
kubectl delete -f namespaces/
```

Or delete them explicitly:

```bash
kubectl delete namespace dev test uat dev-infra test-infra uat-infra
```

Wait for deletion to complete:

```bash
kubectl wait --for=delete namespace/dev namespace/test namespace/uat \
  namespace/dev-infra namespace/test-infra namespace/uat-infra \
  --timeout=120s
```

If a namespace remains in `Terminating`, inspect its resources and finalizers before forcing anything:

```bash
kubectl describe namespace <namespace>
kubectl get all,pvc,secret,configmap -n <namespace>
```

## 7. Verify Cleanup

```bash
helm list -A
kubectl get namespaces
kubectl get pods,svc,pvc -A
```

The repository namespaces and their workloads should no longer appear. Cluster-level resources such as Rancher, Traefik, and other unrelated namespaces should remain untouched.


## 7. Reinstall From Scratch

To recreate the infrastructure after cleanup:

```bash
kubectl apply -f namespaces/

cp environments/dev/mysql.secret.yaml.example \
  environments/dev/mysql.secret.yaml
cp environments/dev/mongodb.secret.yaml.example \
  environments/dev/mongodb.secret.yaml

# Edit the local secret file before continuing.

helm upgrade --install mysql ./helm/infra/mysql \
  --namespace dev-infra \
  --create-namespace \
  --values environments/dev/mysql.yaml \
  --values environments/dev/mysql.secret.yaml

helm upgrade --install mongodb ./helm/infra/mongodb \
  --namespace dev-infra \
  --create-namespace \
  --values environments/dev/mongodb.yaml \
  --values environments/dev/mongodb.secret.yaml

helm upgrade --install kafka ./helm/infra/kafka \
  --namespace dev-infra \
  --create-namespace \
  --values environments/dev/kafka.yaml
```

Repeat the secret-file creation and independent Helm deployments for `test-infra` and `uat-infra` with their matching environment files.

Deploy application services only after the required database namespaces and database Services are ready.
