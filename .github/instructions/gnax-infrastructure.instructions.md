---
description: "Use when editing Helm charts, Kubernetes manifests, environment overlays, values files, or local infrastructure configuration for Rancher Desktop or Kubernetes deployments."
applyTo: "**"
---

# GnaX Local Infrastructure Guidelines

## Operating model

- This repository targets one local Rancher Desktop Kubernetes cluster with three application environments: `dev`, `test`, and `uat`.
- Application services belong in the matching environment namespace: `dev`, `test`, or `uat`.
- Infrastructure components belong in dedicated namespaces: `dev-infra`, `test-infra`, and `uat-infra`.
- Use generic service names such as `orders`, `catalog`, or `notifications`. Do not introduce `frontend` or `backend` as architectural names.
- Deploy each database or application service as an independent Helm release so changing one service does not redeploy unrelated services.

## Repository structure

- Keep infrastructure component chart defaults and metadata under `helm/infra/`.
- Keep application service charts under `helm/services/<service-name>/`.
- Keep infrastructure component overrides in `environments/<env>/mysql.yaml`, `mongodb.yaml`, and `kafka.yaml`.
- Keep each service's overrides in `environments/<env>/<service-name>.yaml`.
- Keep namespace definitions under `namespaces/` and keep them aligned with the namespace values in the database and service charts.
- Preserve the separation between chart defaults and environment-specific values; do not copy environment credentials into chart defaults.
- Keep infrastructure component templates out of `helm/services/`.

## Helm and values conventions

- Prefer chart-level defaults in each chart's `values.yaml` and environment overrides in the matching environment file.
- Use `helm upgrade --install` with a stable release name for repeatable local deployments.
- Use the component name (`mysql`, `mongodb`, or `kafka`) as the release name for infrastructure installations.
- Use the service name as the release name for application services.
- Use consistent names across chart names, release names, Kubernetes Services, and namespace references.
- Keep values structures easy to override without deep duplication.
- Keep credentials in environment values only for this local development setup; use a secret-management solution for shared or production clusters.

## Kubernetes configuration rules

- Keep manifests and values valid and portable for Rancher Desktop.
- Avoid hard-coded environment assumptions; use values-driven namespaces, images, ports, and storage sizes.
- Use `ClusterIP` for internal application and database communication by default.
- Use `NodePort` only when a host IDE or external local tool must connect directly.
- Keep database `Service` names stable: `mysql` and `mongodb`.
- Keep resource names and labels consistent across `dev`, `test`, and `uat`.
- Use `Recreate` for single-replica database deployments that share a `ReadWriteOnce` PVC.
- Do not move a database to another namespace without an explicit backup/restore plan. PVCs are namespace-scoped.

## Setup and deployment workflow

1. Confirm Rancher Desktop Kubernetes is running and the selected context points to the local cluster.
2. Create or reconcile all application and database namespaces with `kubectl apply -f namespaces/`.
3. Validate each infrastructure chart with `helm lint ./helm/infra/<component>` and `helm template` using the target environment's component values.
4. Install or upgrade each infrastructure component independently in the matching `*-infra` namespace.
5. Wait for database pods and verify their Services, PVCs, credentials, and NodePorts.
6. Deploy each application service separately into `dev`, `test`, or `uat` with that service's environment values.
7. Use Kubernetes DNS from services, for example `mongodb.dev-infra.svc.cluster.local` and `mysql.dev-infra.svc.cluster.local`.
8. Use the documented NodePorts only for host-side development tools.

## Validation expectations

- Validate chart changes with `helm lint` and `helm template` for every affected environment.
- Verify rendered namespaces, resource names, ports, NodePorts, and image tags.
- For database changes, verify pod readiness, PVC binding, Service endpoints, and non-root user authentication.
- For service changes, verify deployment readiness, Service selectors, and application-to-database connectivity.
- Prefer minimal, targeted changes that preserve the existing deployment model.

## Editing guidance

- Keep YAML indentation consistent and avoid unrelated restructuring.
- Prefer small, clear edits over broad rewrites.
- When adding values, make their purpose clear from the chart defaults and environment override.
- Update `README.md` when setup commands, namespace names, release names, or connection details change.
- Document significant configuration changes only when they materially reduce ambiguity.
