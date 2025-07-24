# Expert Detail Questions

## Q1: Should the Helm chart create the namespace or assume it exists?
**Default if unknown:** Yes (chart should create namespace for clean deployments)

## Q2: Should ingress TLS be configurable with cert-manager integration support?
**Default if unknown:** Yes (TLS is essential for production deployments)

## Q3: Should the chart support deploying with an existing PVC instead of creating one?
**Default if unknown:** No (chart should manage its own storage for simplicity)

## Q4: Should secrets be embedded in values.yaml or referenced from external secret management?
**Default if unknown:** Both (support inline secrets and external secret references)

## Q5: Should the chart include RBAC resources for service accounts?
**Default if unknown:** No (application doesn't require special permissions)

## Q6: Should the Chrome/Chromium binary path be configurable per deployment?
**Default if unknown:** Yes (different container images may have Chrome in different locations)

## Q7: Should the chart support nodeSelector and tolerations for pod scheduling?
**Default if unknown:** Yes (enterprise deployments often need specific node placement)

## Q8: Should email tracking components be entirely optional with conditional templates?
**Default if unknown:** Yes (many deployments won't need email features)

## Q9: Should the chart support configurable probes (startup, readiness, liveness) timeouts?
**Default if unknown:** Yes (different environments may need different probe settings)

## Q10: Should the chart include NetworkPolicy resources for network security?
**Default if unknown:** No (network policies are environment-specific and often managed separately)