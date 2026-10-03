# Services

Each application is deployed as an independent Helm release into the matching environment namespace:

```text
helm/services/<service-name>/
```

Use generic service names such as `orders`, `catalog`, or `notifications`. Do not place database templates in this directory.

Environment namespaces:

- `dev`
- `test`
- `uat`

Infrastructure namespaces are separate: `dev-infra`, `test-infra`, and `uat-infra`.

Service-specific setup, deployment, and uninstall instructions are in each
service chart's `README.md`. Current chart documentation:

- [GnaX Config Server](./gnax-config-server/README.md), including its Secret

Uninstall a service with `helm uninstall <release-name> --namespace <env>`
after checking the exact release name with `helm list --namespace <env>`.
Charts that manage Secrets alongside an application remove both in the same
release; check each chart README for any separately managed resources.
