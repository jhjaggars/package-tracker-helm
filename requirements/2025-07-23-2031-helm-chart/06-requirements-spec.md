# Helm Chart Requirements Specification

## Problem Statement

The package tracker project currently uses raw Kubernetes manifests for deployment, which lack templating, parameterization, and easy configuration management. A Helm chart is needed to provide:

- **Configurable deployments** without manual manifest editing
- **Reusable templates** for different deployment scenarios 
- **Simplified installation** and upgrade processes
- **Best practice configuration** with sensible defaults

## Solution Overview

Create a comprehensive Helm chart that converts the existing production-ready Kubernetes manifests into templated, configurable resources while preserving all current functionality and security practices.

## Functional Requirements

### FR-1: Core Application Deployment
- **Must** deploy the package tracker application as a single-replica deployment
- **Must** support configurable container image (repository, tag, pullPolicy)
- **Must** maintain existing security context (non-root user, read-only filesystem)
- **Must** preserve existing resource limits and requests with configurability
- **Must** include comprehensive health checks (startup, readiness, liveness probes)

### FR-2: Storage Management
- **Must** create persistent volume claim for SQLite database storage
- **Must** support configurable storage size and storage class
- **Must** support using existing PVC instead of creating new one
- **Must** include tmpfs volume for SQLite temporary files
- **Must** default to 10GB storage size

### FR-3: Configuration Management
- **Must** create ConfigMap with all non-sensitive application settings
- **Must** support all 30+ environment variables from existing manifests
- **Must** provide configurable defaults for server, auto-update, cache, and email settings
- **Must** allow Chrome/Chromium binary path configuration
- **Must** support disabling features via configuration flags

### FR-4: Secrets Management
- **Must** create Secret resource for sensitive API keys and tokens
- **Must** support inline secrets in values.yaml (with encryption tool compatibility)
- **Must** support external secret references for external secret management
- **Must** include all carrier API keys (USPS, UPS, FedEx, DHL)
- **Must** support Gmail OAuth configuration (client ID, secret, tokens)
- **Must** support admin API key configuration

### FR-5: Service and Ingress
- **Must** create ClusterIP service exposing port 8080
- **Must** support optional ingress with configurable hostname
- **Must** support multiple ingress controller annotations beyond NGINX
- **Must** support configurable TLS with cert-manager integration
- **Must** include ingress enabled/disabled toggle

### FR-6: Email Tracking (Optional)
- **Must** support conditional email tracking configuration
- **Must** allow disabling email components entirely
- **Must** configure Gmail OAuth settings when enabled
- **Must** support email scanning parameters (days, intervals, dry-run mode)

### FR-7: Advanced Scheduling
- **Must** support nodeSelector configuration
- **Must** support tolerations for pod scheduling
- **Must** support affinity rules configuration
- **Must** maintain single replica constraint (SQLite limitation)

### FR-8: Namespace Management
- **Must** create namespace for the application
- **Must** support configurable namespace name
- **Must** apply consistent labeling across all resources

## Technical Requirements

### TR-1: Template Structure
```
package-tracker/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── pvc.yaml
│   ├── ingress.yaml (conditional)
│   └── NOTES.txt
└── .helmignore
```

### TR-2: Values.yaml Structure
Based on existing configuration patterns:

```yaml
# Image configuration
image:
  repository: package-tracker
  tag: latest
  pullPolicy: Always

# Resource management
resources:
  limits:
    memory: "512Mi"
    cpu: "500m"
  requests:
    memory: "256Mi"
    cpu: "250m"

# Storage configuration
persistence:
  enabled: true
  existingClaim: ""
  storageClass: ""
  size: 10Gi

# Ingress configuration
ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts: []
  tls: []

# Application configuration
config:
  server:
    host: "0.0.0.0"
    port: 8080
  database:
    path: "/data/database.db"
    emailStatePath: "/data/email-state.db"
  # ... (30+ configuration options)

# Secrets configuration
secrets:
  uspsApiKey: ""
  upsClientId: ""
  # ... (carrier and OAuth secrets)

# Email tracking (optional)
email:
  enabled: false
  # ... (email-specific configuration)

# Scheduling
nodeSelector: {}
tolerations: []
affinity: {}
```

### TR-3: File Conversion Requirements

#### Source to Template Mapping
- `manifests/namespace.yaml` → `templates/namespace.yaml`
- `manifests/deployment.yaml` → `templates/deployment.yaml` 
- `manifests/service.yaml` → `templates/service.yaml`
- `manifests/configmap.yaml` → `templates/configmap.yaml`
- `manifests/secret-template.yaml` → `templates/secret.yaml`
- `manifests/pvc.yaml` → `templates/pvc.yaml`
- `manifests/ingress.yaml` → `templates/ingress.yaml`

#### Template Patterns Required
- Conditional resource creation (`{{- if .Values.ingress.enabled }}`)
- Configuration templating (`{{ .Values.config.server.host }}`)
- Resource naming (`{{ include "package-tracker.fullname" . }}`)
- Label management (`{{ include "package-tracker.labels" . }}`)
- Secret handling (inline vs external references)

### TR-4: Probe Configuration
**Must** support configurable health check settings:
```yaml
probes:
  startup:
    initialDelaySeconds: 10
    periodSeconds: 10
    timeoutSeconds: 3
    failureThreshold: 30
  readiness:
    initialDelaySeconds: 5
    periodSeconds: 10
    timeoutSeconds: 3
    failureThreshold: 3
  liveness:
    initialDelaySeconds: 30
    periodSeconds: 30
    timeoutSeconds: 5
    failureThreshold: 3
```

## Implementation Hints

### Critical Template Patterns

#### 1. Conditional Ingress
```yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
# ... ingress configuration
{{- end }}
```

#### 2. Storage Flexibility
```yaml
{{- if .Values.persistence.existingClaim }}
  claimName: {{ .Values.persistence.existingClaim }}
{{- else }}
  claimName: {{ include "package-tracker.fullname" . }}-data
{{- end }}
```

#### 3. Multi-Controller Ingress Support
```yaml
annotations:
  {{- with .Values.ingress.annotations }}
    {{- toYaml . | nindent 4 }}
  {{- end }}
```

#### 4. Email Feature Toggle
```yaml
{{- if .Values.email.enabled }}
- name: EMAIL_CHECK_INTERVAL
  value: {{ .Values.email.checkInterval | quote }}
{{- end }}
```

### Helper Templates Required
- `package-tracker.name`: Chart name
- `package-tracker.fullname`: Full resource name 
- `package-tracker.chart`: Chart name and version
- `package-tracker.labels`: Standard labels
- `package-tracker.selectorLabels`: Selector labels

### Files to Reference During Implementation
- `/home/jhjaggars/code/package-tracker/manifests/deployment.yaml:36-255` - Environment variable configuration
- `/home/jhjaggars/code/package-tracker/manifests/configmap.yaml:10-60` - Application configuration structure
- `/home/jhjaggars/code/package-tracker/manifests/secret-template.yaml` - Secret key structure
- `/home/jhjaggars/code/package-tracker/manifests/ingress.yaml:9-30` - Ingress annotation patterns

## Acceptance Criteria

### AC-1: Chart Installation
- `helm install package-tracker ./package-tracker` creates working deployment
- All pods reach Ready state within 5 minutes
- Health check endpoint `/api/health` returns 200 OK
- Application serves web UI on configured port

### AC-2: Configuration Flexibility
- Storage size can be changed via `--set persistence.size=20Gi`
- Ingress can be enabled with custom hostname
- Resource limits can be adjusted per environment
- Email tracking can be completely disabled

### AC-3: Secret Management
- Secrets can be provided inline in values.yaml
- External secret references work correctly
- Missing optional secrets don't prevent startup
- Required secrets cause helpful error messages

### AC-4: Upgrade Safety
- `helm upgrade` preserves persistent data
- Configuration changes apply without data loss
- Rolling updates work correctly with single replica
- Helm rollback functions properly

### AC-5: Production Readiness
- Security context matches existing manifests
- Resource limits prevent resource exhaustion
- Health checks prevent traffic to unhealthy pods
- TLS configuration works with cert-manager

## Assumptions

- **Single Replica**: Chart assumes single replica due to SQLite constraints
- **Container Image**: Assumes `package-tracker:latest` is available in registry
- **Storage Class**: Assumes cluster has default storage class if none specified
- **Ingress Controller**: Assumes NGINX by default but supports others via annotations
- **Chrome Dependency**: Assumes container image includes Chromium browser
- **Email Features**: Assumes Gmail OAuth setup is done externally when email tracking enabled
- **Network Access**: Assumes cluster allows outbound HTTPS for carrier API calls
- **Persistent Storage**: Assumes ReadWriteOnce access mode is sufficient for SQLite

This specification provides a comprehensive foundation for implementing a production-ready Helm chart that maintains full compatibility with the existing Kubernetes manifests while adding extensive configurability and deployment flexibility.