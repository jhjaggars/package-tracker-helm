# Helm Chart Testing Plan - Package Tracker

## Overview

This comprehensive testing plan follows Test-Driven Development (TDD) principles to ensure the package tracker Helm chart meets all functional requirements (FR-1 through FR-8) and acceptance criteria (AC-1 through AC-5). The plan covers unit testing for individual templates, integration testing for complete deployments, edge cases, security validation, and upgrade scenarios.

## Testing Framework and Tools

### Primary Testing Tools
- **helm unittest**: Unit testing for Helm templates
- **conftest**: Policy and security testing using OPA
- **kubeval**: Kubernetes YAML validation
- **helm test**: Integration testing with test pods
- **bats**: Bash Automated Testing System for CLI scenarios
- **yamllint**: YAML syntax and formatting validation

### Test Environment Requirements
- Kubernetes cluster (minikube, kind, or cloud cluster)
- Helm 3.x installed
- kubectl configured
- Docker for building test images
- cert-manager (for TLS testing)

## 1. Unit Testing for Individual Helm Templates

### 1.1 Template Rendering Tests

#### Test Suite: `tests/unit/template-rendering_test.yaml`

```yaml
# Basic template rendering validation
suite: test template rendering
templates:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
  - secret.yaml
  - pvc.yaml
  - namespace.yaml
  - ingress.yaml
tests:
  - it: should render deployment with default values
    template: deployment.yaml
    asserts:
      - isKind:
          of: Deployment
      - equal:
          path: metadata.name
          value: RELEASE-NAME-package-tracker
      - equal:
          path: spec.replicas
          value: 1
```

#### Test Cases:

**TC-UT-001: Deployment Template Validation**
- **Purpose**: Verify deployment template renders correctly with various configurations
- **Test Data**:
  ```yaml
  image:
    repository: package-tracker
    tag: v1.0.0
    pullPolicy: IfNotPresent
  resources:
    limits:
      memory: 1Gi
      cpu: 1000m
  ```
- **Expected Outcome**: Deployment manifest with correct image, resources, and replica count
- **Assertions**:
  - Container image matches specified repository and tag
  - Resource limits and requests are applied
  - Security context is non-root with read-only filesystem
  - Single replica is enforced

**TC-UT-002: ConfigMap Template Validation**
- **Purpose**: Verify ConfigMap contains all required environment variables
- **Test Data**: All 30+ configuration variables from requirements
- **Expected Outcome**: ConfigMap with properly templated configuration values
- **Assertions**:
  - All required config keys are present
  - Values are properly quoted and escaped
  - Feature flags are correctly templated
  - Chrome/Chromium path is configurable

**TC-UT-003: Secret Template Validation**
- **Purpose**: Verify secrets are correctly templated for both inline and external references
- **Test Data**:
  ```yaml
  secrets:
    uspsApiKey: "inline-key-value"
    externalSecret:
      name: "external-secret"
      key: "api-key"
  ```
- **Expected Outcome**: Secret manifest with proper data handling
- **Assertions**:
  - Inline secrets are base64 encoded
  - External secret references are properly structured
  - Optional secrets don't break template rendering

**TC-UT-004: Storage Template Validation**
- **Purpose**: Verify PVC creation and existing claim support
- **Test Data**:
  ```yaml
  # Test 1: New PVC creation
  persistence:
    enabled: true
    size: 20Gi
    storageClass: fast-ssd
  
  # Test 2: Existing PVC usage
  persistence:
    enabled: true
    existingClaim: existing-data-claim
  ```
- **Expected Outcome**: Correct PVC configuration or existing claim reference
- **Assertions**:
  - PVC is created when existingClaim is empty
  - Existing claim is referenced when specified
  - Storage size and class are properly applied
  - tmpfs volume is included in deployment

### 1.2 Conditional Rendering Tests

**TC-UT-005: Ingress Conditional Rendering**
- **Purpose**: Verify ingress is only created when enabled
- **Test Data**:
  ```yaml
  # Test 1: Ingress disabled
  ingress:
    enabled: false
  
  # Test 2: Ingress enabled
  ingress:
    enabled: true
    hosts:
      - host: package-tracker.example.com
        paths:
          - path: /
            pathType: Prefix
  ```
- **Expected Outcome**: Ingress manifest only when enabled
- **Assertions**:
  - No ingress resource when disabled
  - Proper ingress resource when enabled
  - TLS configuration applied correctly
  - Multiple ingress controller annotations supported

**TC-UT-006: Email Feature Toggle**
- **Purpose**: Verify email-related configuration is conditional
- **Test Data**:
  ```yaml
  # Test 1: Email disabled
  email:
    enabled: false
  
  # Test 2: Email enabled
  email:
    enabled: true
    checkInterval: "1h"
    gmailOauth:
      clientId: "test-client-id"
  ```
- **Expected Outcome**: Email environment variables only when enabled
- **Assertions**:
  - Email config absent when disabled
  - Email config present when enabled
  - Gmail OAuth secrets properly referenced

### 1.3 Label and Naming Tests

**TC-UT-007: Resource Naming Consistency**
- **Purpose**: Verify consistent naming across all resources
- **Test Data**: Various release names and configurations
- **Expected Outcome**: Consistent naming pattern using helper templates
- **Assertions**:
  - All resources use fullname helper
  - Labels are consistent across resources
  - Selector labels match deployment labels

## 2. Integration Testing for Complete Chart Deployment

### 2.1 Full Deployment Tests

#### Test Suite: `tests/integration/deployment_test.bats`

**TC-IT-001: Basic Chart Installation**
- **Purpose**: Verify chart installs successfully with default values
- **Test Procedure**:
  ```bash
  helm install test-release ./package-tracker
  kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=test-release --timeout=300s
  ```
- **Expected Outcome**: All pods reach Ready state within 5 minutes
- **Validation**:
  - Deployment is created and scaled to 1 replica
  - Pod starts successfully and passes health checks
  - Service endpoints are available
  - ConfigMap and Secret are mounted correctly

**TC-IT-002: Health Check Validation**
- **Purpose**: Verify application health endpoints respond correctly
- **Test Procedure**:
  ```bash
  kubectl port-forward svc/test-release-package-tracker 8080:8080 &
  curl -f http://localhost:8080/api/health
  ```
- **Expected Outcome**: Health endpoint returns 200 OK
- **Validation**:
  - Startup probe allows sufficient initialization time
  - Readiness probe correctly identifies ready state
  - Liveness probe maintains pod health

**TC-IT-003: Storage Persistence Validation**
- **Purpose**: Verify persistent storage works correctly
- **Test Procedure**:
  1. Install chart with PVC
  2. Create test data via application API
  3. Delete and recreate pod
  4. Verify data persistence
- **Expected Outcome**: Data survives pod restarts
- **Validation**:
  - PVC is created with correct size and storage class
  - Database file persists across pod restarts
  - tmpfs volumes are properly mounted

### 2.2 Configuration Validation Tests

**TC-IT-004: Custom Values Application**
- **Purpose**: Verify custom values are applied correctly
- **Test Data**:
  ```yaml
  image:
    tag: v2.0.0
  resources:
    limits:
      memory: 2Gi
  persistence:
    size: 50Gi
  config:
    server:
      port: 9090
  ```
- **Expected Outcome**: All custom values reflected in deployed resources
- **Validation**:
  - Container uses specified image tag
  - Resource limits match custom values
  - PVC has custom size
  - Application listens on custom port

**TC-IT-005: Ingress with TLS Testing**
- **Purpose**: Verify ingress works with TLS and cert-manager
- **Test Data**:
  ```yaml
  ingress:
    enabled: true
    className: nginx
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-staging
    hosts:
      - host: test.example.com
        paths:
          - path: /
            pathType: Prefix
    tls:
      - secretName: test-tls
        hosts:
          - test.example.com
  ```
- **Expected Outcome**: HTTPS access works correctly
- **Validation**:
  - Ingress resource is created with correct annotations
  - TLS certificate is issued by cert-manager
  - Application accessible via HTTPS

## 3. Edge Case Testing Scenarios

### 3.1 Resource Constraint Tests

**TC-EC-001: Insufficient Resources**
- **Purpose**: Verify behavior when cluster resources are insufficient
- **Test Procedure**: Set resource requests higher than available cluster capacity
- **Expected Outcome**: Pod remains in Pending state with clear error message
- **Validation**: Events show resource constraint reasons

**TC-EC-002: Storage Class Not Available**
- **Purpose**: Verify behavior when specified storage class doesn't exist
- **Test Procedure**: Specify non-existent storage class
- **Expected Outcome**: PVC remains in Pending state
- **Validation**: Clear error messages about missing storage class

**TC-EC-003: Image Pull Failures**
- **Purpose**: Verify behavior with non-existent container images
- **Test Procedure**: Specify invalid image repository or tag
- **Expected Outcome**: Pod shows ImagePullBackOff status
- **Validation**: Events clearly indicate image pull failure

### 3.2 Configuration Edge Cases

**TC-EC-004: Empty Configuration Values**
- **Purpose**: Verify handling of empty or missing configuration
- **Test Data**: Values file with empty strings and null values
- **Expected Outcome**: Application uses sensible defaults
- **Validation**: ConfigMap contains default values where appropriate

**TC-EC-005: Special Characters in Configuration**
- **Purpose**: Verify proper handling of special characters in config values
- **Test Data**: Configuration with quotes, newlines, and special characters
- **Expected Outcome**: Values are properly escaped and quoted
- **Validation**: Application receives correct configuration values

**TC-EC-006: Malformed Secrets**
- **Purpose**: Verify handling of improperly formatted secrets
- **Test Data**: Invalid base64 data or missing required secret keys
- **Expected Outcome**: Clear error messages during installation
- **Validation**: Helm install fails with descriptive error

## 4. Configuration Validation Testing

### 4.1 Schema Validation Tests

**TC-CV-001: Values Schema Validation**
- **Purpose**: Verify values.yaml conforms to expected schema
- **Test Tool**: JSON Schema validation
- **Test Data**: Various valid and invalid values.yaml files
- **Expected Outcome**: Schema violations are caught before deployment
- **Validation**:
  - Required fields are enforced
  - Data types are validated
  - Enum values are restricted properly

**TC-CV-002: Kubernetes Manifest Validation**
- **Purpose**: Verify generated manifests are valid Kubernetes YAML
- **Test Tool**: kubeval
- **Test Procedure**:
  ```bash
  helm template ./package-tracker | kubeval
  ```
- **Expected Outcome**: All generated manifests pass Kubernetes validation
- **Validation**: No schema violations in any generated resource

### 4.2 Configuration Consistency Tests

**TC-CV-003: Cross-Reference Validation**
- **Purpose**: Verify configuration references between resources are consistent
- **Test Cases**:
  - Service selector matches deployment labels
  - Volume mounts reference existing volumes
  - Secret references point to created secrets
- **Expected Outcome**: All cross-references are valid
- **Validation**: No dangling references or mismatched selectors

## 5. Security Testing Requirements

### 5.1 Security Context Validation

**TC-SEC-001: Non-Root User Enforcement**
- **Purpose**: Verify containers run as non-root user
- **Test Procedure**: Check pod security context
- **Expected Outcome**: Container runs as UID > 0
- **Validation**:
  ```bash
  kubectl exec pod-name -- id
  # Should return uid != 0
  ```

**TC-SEC-002: Read-Only Filesystem**
- **Purpose**: Verify root filesystem is read-only
- **Test Procedure**: Attempt to write to root filesystem
- **Expected Outcome**: Write operations fail
- **Validation**:
  ```bash
  kubectl exec pod-name -- touch /test-file
  # Should fail with permission denied
  ```

**TC-SEC-003: Capability Dropping**
- **Purpose**: Verify unnecessary capabilities are dropped
- **Test Tool**: Conftest with security policies
- **Expected Outcome**: Only required capabilities are granted
- **Validation**: Security policy checks pass

### 5.2 Secret Security Tests

**TC-SEC-004: Secret Encryption at Rest**
- **Purpose**: Verify secrets are properly encrypted
- **Test Procedure**: Check etcd directly for secret data
- **Expected Outcome**: Secret data is encrypted, not plain text
- **Validation**: etcd data shows encrypted values

**TC-SEC-005: Secret Access Controls**
- **Purpose**: Verify proper RBAC for secret access
- **Test Procedure**: Test secret access with different service accounts
- **Expected Outcome**: Only authorized pods can access secrets
- **Validation**: Unauthorized access attempts fail

### 5.3 Network Security Tests

**TC-SEC-006: Network Policy Compatibility**
- **Purpose**: Verify chart works with network policies
- **Test Procedure**: Deploy with restrictive network policies
- **Expected Outcome**: Application functions with proper network access
- **Validation**: Required network flows work, others are blocked

## 6. Upgrade and Rollback Testing

### 6.1 Upgrade Scenarios

**TC-UP-001: Configuration-Only Upgrade**
- **Purpose**: Verify upgrades with only configuration changes
- **Test Procedure**:
  1. Install chart with initial configuration
  2. Upgrade with modified configuration values
  3. Verify new configuration is applied
- **Expected Outcome**: Configuration changes applied without data loss
- **Validation**:
  - ConfigMap is updated
  - Pod restarts with new configuration
  - Persistent data remains intact

**TC-UP-002: Image Version Upgrade**
- **Purpose**: Verify application image upgrades
- **Test Procedure**:
  1. Install chart with version 1.0.0
  2. Upgrade to version 2.0.0
  3. Verify application version and functionality
- **Expected Outcome**: New image deployed successfully
- **Validation**:
  - Rolling update completes successfully
  - Application serves requests during upgrade
  - Database schema migrations (if any) complete

**TC-UP-003: Resource Limit Changes**
- **Purpose**: Verify resource limit modifications during upgrade
- **Test Procedure**:
  1. Install with initial resource limits
  2. Upgrade with increased limits
  3. Verify pod gets new resource allocation
- **Expected Outcome**: Pod restarts with new resource limits
- **Validation**: kubectl describe shows updated resource limits

### 6.2 Rollback Scenarios

**TC-RB-001: Configuration Rollback**
- **Purpose**: Verify rollback after configuration changes
- **Test Procedure**:
  1. Install chart
  2. Upgrade with problematic configuration
  3. Roll back to previous version
- **Expected Outcome**: Previous configuration is restored
- **Validation**:
  - ConfigMap reverts to previous values
  - Application functions correctly
  - No data corruption

**TC-RB-002: Failed Upgrade Rollback**
- **Purpose**: Verify rollback after failed upgrade
- **Test Procedure**:
  1. Attempt upgrade with invalid configuration
  2. Rollback when upgrade fails
- **Expected Outcome**: System returns to previous working state
- **Validation**:
  - All resources match previous revision
  - Application remains available
  - Data integrity maintained

### 6.3 Data Persistence During Upgrades

**TC-DP-001: Database Integrity**
- **Purpose**: Verify database integrity across upgrades
- **Test Procedure**:
  1. Create test data in application
  2. Perform multiple upgrade/rollback cycles
  3. Verify data consistency
- **Expected Outcome**: Database remains consistent and accessible
- **Validation**:
  - All test data remains available
  - Database file is not corrupted
  - Application queries return expected results

## Test Execution Strategy

### Phase 1: Development Testing
1. **Unit Tests**: Run during template development
   - Execute after each template modification
   - Automated via pre-commit hooks
   - Required for all pull requests

2. **Static Analysis**: Continuous validation
   - yamllint for YAML formatting
   - kubeval for Kubernetes schema validation
   - conftest for security policy checks

### Phase 2: Integration Testing
1. **Local Testing**: Developer workstation
   - minikube or kind cluster
   - Basic functionality validation
   - Configuration testing

2. **CI/CD Pipeline**: Automated testing
   - Matrix testing across Kubernetes versions
   - Multiple ingress controller testing
   - Storage class compatibility testing

### Phase 3: End-to-End Testing
1. **Staging Environment**: Production-like testing
   - Full security context testing
   - Performance validation
   - Upgrade/rollback scenarios

2. **Production Validation**: Final verification
   - Canary deployments
   - Monitoring and alerting verification
   - Disaster recovery testing

## Test Data Management

### Configuration Test Data

#### Minimal Configuration
```yaml
# Absolute minimum required values
image:
  repository: package-tracker
  tag: latest
```

#### Complete Configuration
```yaml
# Full featured configuration for comprehensive testing
image:
  repository: package-tracker
  tag: v1.0.0
  pullPolicy: IfNotPresent

resources:
  limits:
    memory: 1Gi
    cpu: 1000m
  requests:
    memory: 512Mi
    cpu: 500m

persistence:
  enabled: true
  size: 20Gi
  storageClass: fast-ssd

ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: package-tracker.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: package-tracker-tls
      hosts:
        - package-tracker.example.com

config:
  server:
    host: "0.0.0.0"
    port: 8080
    logLevel: "info"
  database:
    path: "/data/database.db"
    emailStatePath: "/data/email-state.db"
  autoUpdate:
    enabled: true
    interval: "1h"
  cache:
    enabled: true
    size: "100MB"

secrets:
  adminApiKey: "super-secret-admin-key"
  uspsApiKey: "usps-test-key"
  upsClientId: "ups-client-id"
  upsClientSecret: "ups-client-secret"
  fedexApiKey: "fedex-test-key"
  fedexSecretKey: "fedex-secret-key"
  dhlApiKey: "dhl-test-key"

email:
  enabled: true
  checkInterval: "30m"
  scanDays: 7
  dryRun: false
  gmailOauth:
    clientId: "gmail-client-id"
    clientSecret: "gmail-client-secret"
    refreshToken: "gmail-refresh-token"

nodeSelector:
  kubernetes.io/arch: amd64

tolerations:
  - key: "node-type"
    operator: "Equal"
    value: "compute"
    effect: "NoSchedule"

affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
            - key: "node-type"
              operator: "In"
              values: ["compute"]
```

#### Invalid Configuration Test Cases
```yaml
# Test case: Invalid resource format
resources:
  limits:
    memory: "invalid-format"
    cpu: -100

# Test case: Missing required image
image: {}

# Test case: Invalid ingress configuration
ingress:
  enabled: true
  hosts: "invalid-format-should-be-array"
```

## Success Criteria

### Functional Requirements Coverage
- **FR-1**: ✅ Core application deployment tests (TC-IT-001, TC-UT-001)
- **FR-2**: ✅ Storage management tests (TC-UT-004, TC-IT-003)
- **FR-3**: ✅ Configuration management tests (TC-UT-002, TC-IT-004)
- **FR-4**: ✅ Secrets management tests (TC-UT-003, TC-SEC-004, TC-SEC-005)
- **FR-5**: ✅ Service and ingress tests (TC-IT-005, TC-UT-005)
- **FR-6**: ✅ Email tracking tests (TC-UT-006)
- **FR-7**: ✅ Advanced scheduling tests (TC-IT-004 with nodeSelector/affinity)
- **FR-8**: ✅ Namespace management tests (TC-UT-007)

### Acceptance Criteria Coverage
- **AC-1**: ✅ Chart installation tests (TC-IT-001, TC-IT-002)
- **AC-2**: ✅ Configuration flexibility tests (TC-IT-004, TC-UT-005)
- **AC-3**: ✅ Secret management tests (TC-UT-003, TC-EC-006)
- **AC-4**: ✅ Upgrade safety tests (TC-UP-001, TC-UP-002, TC-RB-001, TC-DP-001)
- **AC-5**: ✅ Production readiness tests (TC-SEC-001, TC-SEC-002, TC-IT-002)

### Test Coverage Metrics
- **Template Coverage**: 100% of all templates tested
- **Configuration Coverage**: All values.yaml options tested
- **Edge Case Coverage**: Major failure modes tested
- **Security Coverage**: All security contexts validated
- **Upgrade Coverage**: All upgrade/rollback scenarios tested

## Continuous Integration Integration

### Pre-commit Hooks
```bash
#!/bin/bash
# .git/hooks/pre-commit
set -e

echo "Running Helm chart tests..."

# Unit tests
helm unittest package-tracker/

# Template validation
helm template ./package-tracker | kubeval

# Security policy checks
helm template ./package-tracker | conftest test --policy security-policies -

# YAML linting
yamllint package-tracker/

echo "All tests passed!"
```

### CI/CD Pipeline Configuration
```yaml
# .github/workflows/helm-test.yml
name: Helm Chart Testing
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        k8s-version: [1.24, 1.25, 1.26, 1.27]
    steps:
      - uses: actions/checkout@v3
      - name: Setup Helm
        uses: azure/setup-helm@v3
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: v${{ matrix.k8s-version }}.0
      - name: Create kind cluster
        uses: helm/kind-action@v1.4.0
        with:
          kubernetes_version: v${{ matrix.k8s-version }}.0
      - name: Run helm unittest
        run: helm unittest package-tracker/
      - name: Install and test chart
        run: |
          helm install test-release ./package-tracker
          kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=test-release --timeout=300s
          helm test test-release
```

This comprehensive testing plan ensures thorough validation of the Helm chart implementation against all requirements while following TDD principles and providing clear, actionable test cases for development and continuous integration.