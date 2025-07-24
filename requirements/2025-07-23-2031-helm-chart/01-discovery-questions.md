# Discovery Questions

## Q1: Will the Helm chart need to support multiple deployment environments (dev, staging, prod)?
**Default if unknown:** Yes (standard practice for enterprise deployments)

## Q2: Should the Helm chart include ingress configuration for external access?
**Default if unknown:** Yes (based on existing ingress.yaml manifest)

## Q3: Will users need to customize storage class and size for the persistent volume?
**Default if unknown:** Yes (different environments have different storage requirements)

## Q4: Should the chart support configuring multiple replicas for high availability?
**Default if unknown:** No (current app uses SQLite which doesn't support multiple replicas)

## Q5: Will the chart need to handle sensitive configuration like API keys and tokens?
**Default if unknown:** Yes (based on extensive secret configuration in existing manifests)

## Q6: Should the chart include optional monitoring and observability components?
**Default if unknown:** No (keeping chart focused on core application functionality)

## Q7: Will users need to customize resource limits and requests per environment?
**Default if unknown:** Yes (different environments have different resource constraints)

## Q8: Should the chart support external database configuration as an alternative to SQLite?
**Default if unknown:** No (current architecture is SQLite-based with persistent storage)

## Q9: Will the chart need to support custom ingress controllers beyond NGINX?
**Default if unknown:** Yes (enterprise environments often use different ingress controllers)

## Q10: Should the chart include email tracking configuration as optional components?
**Default if unknown:** Yes (email tracking is a major feature with complex configuration)