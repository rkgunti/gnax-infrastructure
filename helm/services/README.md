# Services

Each application is deployed as an independent Helm release. Keep chart
defaults and templates in the service chart, and put active overrides under
`values/services/`:

```text
helm/services/<service-name>/
```

Use generic service names such as `orders`, `catalog`, or `notifications`. Do not place database templates in this directory.

The current local application namespace is `dev`; infrastructure is deployed
separately into `dev-infra`. This branch uses one shared values set. Environment
branches can carry their own overrides later without changing the service
chart layout.

Service-specific setup, deployment, and uninstall instructions are in each
service chart's `README.md`. Current chart documentation:

- [GnaX Config Server](./gnax-config-server/README.md), including its Secret
- [GnaX Identity Service](./gnax-identity-service/README.md), including
  file-based JWT Secret inputs

Uninstall a service with `helm uninstall <release-name> --namespace dev` after
checking the exact release name with `helm list --namespace dev`.
Charts that manage Secrets alongside an application remove both in the same
release; check each chart README for any separately managed resources.
