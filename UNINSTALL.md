# Uninstall and Cleanup

This document removes the platform resources managed by this repository from the local Rancher Desktop Kubernetes cluster.

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
dev-db
test-db
uat-db
```

Do not delete a namespace if it contains resources managed outside this repository.

## 3. Remove Application Service Releases

Application services are installed into `dev`, `test`, or `uat`. List releases in each namespace:

```bash
helm list -n dev
helm list -n test
helm list -n uat
```

Uninstall only the service releases that belong to this platform:

```bash
helm uninstall <service-name> --namespace dev
helm uninstall <service-name> --namespace test
helm uninstall <service-name> --namespace uat
```

Repeat for every service release in each environment. Do not uninstall unrelated releases such as cluster ingress or Rancher-managed components.

If services were installed with a different release name, use the name shown by `helm list`.

## 4. Remove Database Releases

The current migrated releases may still be named `platform`, while new installations should use `databases`. Check first:

```bash
helm list -n dev-db
helm list -n test-db
helm list -n uat-db
```

Uninstall the database release found in each database namespace:

```bash
helm uninstall platform --namespace dev-db
helm uninstall platform --namespace test-db
helm uninstall platform --namespace uat-db
```

For releases created with the newer name, use:

```bash
helm uninstall databases --namespace dev-db
helm uninstall databases --namespace test-db
helm uninstall databases --namespace uat-db
```

Only run the command for a release that exists. A missing release is harmless; do not uninstall both names unless both are listed.

## 5. Decide What to Do With Database PVCs

Inspect remaining PVCs:

```bash
kubectl get pvc -n dev-db
kubectl get pvc -n test-db
kubectl get pvc -n uat-db
```

If database data must be preserved, stop here and create a backup before deleting PVCs.

For a complete disposable local reset, delete the database PVCs:

```bash
kubectl delete pvc --all -n dev-db
kubectl delete pvc --all -n test-db
kubectl delete pvc --all -n uat-db
```

Deleting PVCs permanently removes the local database data when the underlying storage is reclaimed.

## 6. Remove Repository Namespaces

After all service and database releases are removed, delete the six namespaces:

```bash
kubectl delete -f namespaces/
```

Or delete them explicitly:

```bash
kubectl delete namespace dev test uat dev-db test-db uat-db
```

Wait for deletion to complete:

```bash
kubectl wait --for=delete namespace/dev namespace/test namespace/uat \
  namespace/dev-db namespace/test-db namespace/uat-db \
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

To recreate the platform after cleanup:

```bash
kubectl apply -f namespaces/

cp environments/dev/databases.secret.yaml.example \
   environments/dev/databases.secret.yaml

# Edit the local secret file before continuing.

helm upgrade --install databases ./helm/databases \
  --namespace dev-db \
  --create-namespace \
  --values environments/dev/databases.yaml \
  --values environments/dev/databases.secret.yaml
```

Repeat the secret-file creation and Helm deployment for `test-db` and `uat-db` with their matching environment files.

Deploy application services only after the required database namespaces and database Services are ready.
