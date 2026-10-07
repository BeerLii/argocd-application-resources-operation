# Argo CD application resource operation WorkflowTemplate

`workflowtemplate.yaml` defines a WorkflowTemplate that performs selected-resource
sync (called `resync` in the payload) or delete through the Argo CD API. The
Application is a separate WorkflowTemplate input; it is not repeated in the payload.
It never syncs/deletes the entire application unless the caller explicitly lists every resource.

## Payload

```json
{
  "operations": [
    {
      "operation": "resync",
      "resources": [
        {
          "group": "apps",
          "version": "v1",
          "kind": "Deployment",
          "namespace": "payments",
          "name": "payments-api"
        }
      ]
    },
    {
      "operation": "delete",
      "resources": [
        {
          "group": "",
          "version": "v1",
          "kind": "Service",
          "namespace": "payments",
          "name": "payments-api-old"
        }
      ]
    }
  ],
  "options": {
    "insecureTLS": false
  }
}
```

- WorkflowTemplate input `application` is required and contains the Argo CD Application name.
- Payload `operations` is a non-empty array; each item has `operation` (`resync`/`sync` or `delete`) and its own `resources` array. This allows resync and delete in one payload.
- `resources` is a non-empty list. `group` is empty for core API resources; `version` is normally `v1`.
- `namespace` may be empty for cluster-scoped resources.
- `options.insecureTLS` is optional and should only be enabled for a deliberately trusted test endpoint.

Example submission with both `resync` and `delete` in one payload:

```bash
argo submit --from workflowtemplate/argocd-application-resource-operation \
  -p env=prod \
  -p application=payments-api \
  -p payload='{"operations":[{"operation":"resync","resources":[{"group":"apps","version":"v1","kind":"Deployment","namespace":"payments","name":"payments-api"}]},{"operation":"delete","resources":[{"group":"","version":"v1","kind":"Service","namespace":"payments","name":"payments-api-old"}]}]}' \
  -n argo
```

## Required environment configuration

The template expects this ConfigMap and Secret (keys must match the `env` parameter):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-resource-operation-endpoints
  namespace: argo
data:
  endpoints.json: |
    {"dev":"https://argocd-dev.example.com","prod":"https://argocd.example.com"}
---
apiVersion: v1
kind: Secret
metadata:
  name: argocd-resource-operation-tokens
  namespace: argo
type: Opaque
stringData:
  dev: replace-with-a-least-privilege-token
  prod: replace-with-a-least-privilege-token
```

The token should have only the required Argo CD application/resource permissions.
Pin the Python image digest and use a mounted CA bundle in production when the Argo
CD endpoint uses a private CA; avoid `insecureTLS` outside testing.
