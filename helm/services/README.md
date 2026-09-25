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

Platform namespaces are separate: `dev-platform`, `test-platform`, and `uat-platform`.
