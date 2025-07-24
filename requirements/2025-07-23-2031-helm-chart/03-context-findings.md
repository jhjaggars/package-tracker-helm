# Context Findings

## Existing Infrastructure Analysis

### Current State
- **No existing Helm charts** in the package-tracker codebase
- **Complete Kubernetes manifests** already exist in `/home/jhjaggars/code/package-tracker/manifests/`
- **Well-documented deployment** with comprehensive README (309 lines)
- **Production-ready configuration** with security best practices

### Existing Kubernetes Manifests Structure

#### Core Resources
1. **namespace.yaml** - Creates `package-tracker` namespace
2. **deployment.yaml** - Main application deployment with 30+ environment variables
3. **service.yaml** - ClusterIP service exposing port 8080
4. **configmap.yaml** - Non-sensitive configuration (server, auto-update, cache, email settings)
5. **secret-template.yaml** - Template for API keys (USPS, UPS, FedEx, DHL, Gmail OAuth)
6. **pvc.yaml** - 10GB persistent volume for SQLite databases
7. **ingress.yaml** - External access with NGINX annotations

#### Container Configuration
- **Image:** `package-tracker:latest` 
- **Multi-stage build:** Node.js frontend + Go backend
- **Security:** Non-root user (1001), read-only filesystem, no privilege escalation
- **Dependencies:** Chromium for web scraping, SQLite for data persistence
- **Health checks:** Startup, readiness, liveness probes on `/api/health`

#### Resource Management
- **Single replica** (SQLite constraint)
- **Limits:** 512Mi memory, 500m CPU
- **Requests:** 256Mi memory, 250m CPU
- **Storage:** 10GB persistent volume + tmpfs for temp files

### Application Architecture Insights

#### Configuration Categories
1. **Server Settings:** Host, port, database paths
2. **Auto-Update Engine:** Batch processing, timeouts, per-carrier settings
3. **Caching System:** TTL configuration, disable flags
4. **Email Tracking:** Gmail OAuth, scan intervals, search parameters
5. **Carrier Integrations:** API keys for USPS, UPS, FedEx, DHL
6. **Web Scraping:** Chrome/Chromium configuration
7. **Security:** Admin API authentication

#### Key Technical Constraints
- **SQLite dependency** prevents horizontal scaling
- **Persistent storage required** for database files
- **Chrome browser needed** for carrier web scraping
- **Complex environment configuration** (30+ variables)

### Helm Chart Conversion Requirements

#### Template Structure Needed
```
templates/
├── namespace.yaml
├── deployment.yaml
├── service.yaml
├── configmap.yaml
├── secret.yaml
├── pvc.yaml
├── ingress.yaml (conditional)
└── NOTES.txt
```

#### Values.yaml Structure Required
Based on existing configuration patterns:
- **Image configuration** (repository, tag, pullPolicy)
- **Resource management** (limits, requests)
- **Storage configuration** (size, storageClass)
- **Ingress settings** (enabled, host, annotations, TLS)
- **Application config** (server, auto-update, cache, email)
- **Secrets management** (API keys, tokens)
- **Security context** (user, group, filesystem)

#### Critical Conversion Patterns
1. **Conditional ingress** based on `ingress.enabled`
2. **Configurable storage** class and size
3. **Flexible ingress controllers** beyond NGINX
4. **Optional email tracking** components
5. **Environment-specific resource allocation**
6. **Comprehensive secret handling**

### Related Features Found
- **Email tracking workflow** with Gmail OAuth integration
- **Multi-carrier support** (USPS, UPS, FedEx, DHL, Amazon)
- **Auto-update system** with configurable batching
- **Web scraping fallback** when API keys unavailable
- **Admin API** with authentication
- **Caching layer** with TTL management

### Files That Need Templating
All existing manifests will be converted to Helm templates:
- `/home/jhjaggars/code/package-tracker/manifests/deployment.yaml` → `templates/deployment.yaml`
- `/home/jhjaggars/code/package-tracker/manifests/service.yaml` → `templates/service.yaml`
- `/home/jhjaggars/code/package-tracker/manifests/configmap.yaml` → `templates/configmap.yaml`
- `/home/jhjaggars/code/package-tracker/manifests/secret-template.yaml` → `templates/secret.yaml`
- `/home/jhjaggars/code/package-tracker/manifests/pvc.yaml` → `templates/pvc.yaml`
- `/home/jhjaggars/code/package-tracker/manifests/ingress.yaml` → `templates/ingress.yaml`
- `/home/jhjaggars/code/package-tracker/manifests/namespace.yaml` → `templates/namespace.yaml`

### Security Considerations
- **Secret management** with optional external secret providers
- **Non-root execution** with proper user/group settings
- **Read-only filesystem** with necessary writable volumes
- **Network policies** consideration for ingress access
- **RBAC requirements** if service accounts needed