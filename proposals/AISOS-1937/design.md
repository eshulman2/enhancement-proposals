# Technical Specification

**Document Version**: 1.0
**Date**: 2026-06-25
**Status**: Draft
**Parent PRD**: AISOS-1937

---

## 1. Overview

This technical specification defines the schema integration, validation behavior, resolution fallback chain, and downstream template mapping for the optional `bootstrapFlavor` property within the OpenStack platform configuration of the OpenShift Installer. It establishes the testable behavioral contracts for decoupling the temporary bootstrap node flavor from the control plane node flavor.

### 1.1 Tiger the Cat

Once upon a time, in a cozy data center, lived a cat named Tiger. Tiger was a curious tabby with stripes that resembled the copper cabling in a neatly organized server rack. While the engineers spent their days debugging OpenStack configurations and resolving provisioning timeouts, Tiger had a much more critical job: keeping watch over the warm exhaust of the primary bootstrap node. One afternoon, when a particularly heavy installation process commenced, Tiger curled up on top of the chassis, purring in perfect sync with the cooling fans. His presence was a comforting reminder that even the most complex cloud automation systems benefit from a little warmth, patience, and a watchful eye.

---

## 2. User Scenarios

### Priority Legend
- **P1**: Critical path — must work for MVP
- **P2**: Important — required for full release
- **P3**: Enhancement — can be deferred

### 2.1 P1 Scenarios (Critical)

#### SC-001: Explicit Bootstrap Flavor Provisioning
**Preconditions**: The target OpenStack cloud is reachable, the credentials are valid, and the configured bootstrap flavor exists in the OpenStack Nova flavor registry.  
**Trigger**: Administrator executes `openshift-install create cluster` with an explicit `bootstrapFlavor` specified in `install-config.yaml`.  

**Acceptance Criteria**:
```gherkin
Given a valid install-config.yaml containing "platform.openstack.bootstrapFlavor" set to "m1.large"
  And "controlPlane.platform.openstack.type" set to "m1.xlarge-pinned"
  And the flavor "m1.large" is active on the target OpenStack cloud
When the administrator runs the installer to provision the cluster
Then the installer resolves the bootstrap flavor to "m1.large"
  And the temporary bootstrap VM is provisioned in OpenStack using the "m1.large" flavor
  And the control plane master VMs are provisioned using the "m1.xlarge-pinned" flavor
```

**Edge Cases**:
- **Leading/Trailing Whitespace**: If the specified `bootstrapFlavor` contains leading or trailing whitespaces (e.g., `" m1.large "`), the installer parser must trim the whitespace prior to validation and API lookup (traceable to FR-001).

---

#### SC-002: Default Fallback Flavor Inheritance
**Preconditions**: The `platform.openstack.bootstrapFlavor` property is omitted from `install-config.yaml`.  
**Trigger**: Administrator executes `openshift-install create cluster`.  

**Acceptance Criteria**:
```gherkin
Given an install-config.yaml where "platform.openstack.bootstrapFlavor" is omitted
  And "controlPlane.platform.openstack.type" is set to "m1.xlarge-pinned"
When the administrator runs the installer to provision the cluster
Then the installer falls back to the control plane flavor
  And resolves the bootstrap flavor to "m1.xlarge-pinned"
  And provisions the temporary bootstrap VM using the "m1.xlarge-pinned" flavor
```

**Edge Cases**:
- **Control Plane Omitted**: If both `platform.openstack.bootstrapFlavor` and `controlPlane.platform.openstack.type` are omitted, the resolver must fallback to `platform.openstack.defaultMachinePlatform.type` (traceable to FR-003).

---

#### SC-003: Validation of Non-Existent Flavor
**Preconditions**: Target OpenStack cloud compute API is responsive and authenticated.  
**Trigger**: Administrator runs installer pre-flight validation.  

**Acceptance Criteria**:
```gherkin
Given an install-config.yaml containing "platform.openstack.bootstrapFlavor" set to "m1.invalid-flavor"
  And the flavor "m1.invalid-flavor" does not exist in the target OpenStack cloud flavor registry
When the installer pre-flight validation executes
Then the validation fails immediately
  And the installer writes the error "platform.openstack.bootstrapFlavor: Invalid value: \"m1.invalid-flavor\": flavor \"m1.invalid-flavor\" not found" to stdout
  And the installation process aborts with a non-zero exit code
```

---

### 2.2 P2 Scenarios (Important)

#### SC-004: Resource Constrained Flavor Warning
**Preconditions**: Target OpenStack cloud compute API is responsive, and the configured flavor exists but has a low resource specification.  
**Trigger**: Administrator runs installer pre-flight validation.  

**Acceptance Criteria**:
```gherkin
Given an install-config.yaml containing "platform.openstack.bootstrapFlavor" set to "m1.small"
  And the "m1.small" flavor is defined in OpenStack with 2 vCPUs and 8 GB RAM
When the installer pre-flight validation executes
Then the validation emits a warning: "Warning: platform.openstack.bootstrapFlavor \"m1.small\" does not meet the minimum recommended resource profile (4 vCPUs, 16 GB RAM). This may cause bootstrap timeouts or OOM failures."
  And the installer allows the execution to continue without aborting
```

---

## 3. Functional Requirements

### 3.1 Core Functions

| ID | Function | Description | Inputs | Outputs |
|----|----------|-------------|--------|---------|
| FN-001 | `ResolveBootstrapFlavor` | Evaluates the schema hierarchy to determine the target flavor name to use for the bootstrap node. | `config *types.InstallConfig` | `string` (resolved flavor name) |
| FN-002 | `ValidateBootstrapFlavor` | Performs a pre-flight API query against OpenStack Nova service to check if the flavor exists and inspect resource properties. | `client *openstack.Client`, `flavorName string` | `field.ErrorList` |
| FN-003 | `MapBootstrapToTerraform` | Inject the resolved bootstrap flavor name into the Terraform variables map for provisioning. | `flavorName string` | `map[string]interface{}` containing `openstack_bootstrap_flavor` |

### 3.2 Business Rules

| ID | Rule | Condition | Action |
|----|------|-----------|--------|
| BR-001 | **Resolution Fallback Chain** | `platform.openstack.bootstrapFlavor` is empty. | Resolve value in order: <br>1. `controlPlane.platform.openstack.type`<br>2. `platform.openstack.defaultMachinePlatform.type` |
| BR-002 | **Exact String Match** | Resolving and checking flavor on OpenStack Nova API. | The string match against the Nova flavor catalog must be exact and case-sensitive. |
| BR-003 | **Hardware Profile Warn** | Resolved flavor has `< 4 vCPUs` OR `< 16 GB RAM`. | Emit validation warning warning of potential bootstrap failures, but do not block provisioning. |
| BR-004 | **Hardware Profile Fail** | Resolved flavor has `< 1 vCPU` OR `< 4 GB RAM` (hard floor). | Raise a strict validation error and halt installer execution. |

---

## 4. Interface Changes

### 4.1 Configuration Schema

```yaml
# install-config.yaml excerpt
platform:
  openstack:
    # Existing fields omitted for brevity
    bootstrapFlavor: "string - optional - Specific OpenStack flavor for the temporary bootstrap machine"
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `platform.openstack.bootstrapFlavor` | string | No | Null (defaults to control plane) | Distinct OpenStack VM flavor name dedicated specifically to the temporary bootstrap machine. |

---

## 5. Error Handling

### 5.1 Error Scenarios

| Scenario | Error | Resolution |
|----------|-------|------------|
| Specifying a flavor name that does not exist in the OpenStack environment. | `platform.openstack.bootstrapFlavor: Invalid value: "<flavor>": flavor "<flavor>" not found` | Update the `bootstrapFlavor` configuration with a flavor that exists on the OpenStack cloud. |
| Specified flavor exists but is below absolute hard floors (< 1 vCPU or < 4 GB RAM). | `platform.openstack.bootstrapFlavor: Invalid value: "<flavor>": flavor must have at least 1 vCPU and 4 GB RAM` | Select a flavor that meets the minimum bootstrap environment requirements. |
| OpenStack API network timeout or authentication failure during pre-flight lookup. | `platform.openstack.bootstrapFlavor: Internal error: failed to query OpenStack Nova API: <error_detail>` | Verify target cloud credentials and API endpoints reachability in `clouds.yaml`. |

---

## 6. Non-Functional Requirements

| Requirement | Target | Category |
|-------------|--------|----------|
| **Backward Compatibility** (NFR-001) | 100% parsing and installation success for legacy configurations that do not define `bootstrapFlavor`. | Compatibility |
| **Validation Latency** | Pre-flight API check must complete within `< 5 seconds` under nominal network conditions. | Performance |

---

## 7. Testing Requirements

### 7.1 Test Scenarios

| ID | Scenario | Type | Priority | Automated |
|----|----------|------|----------|-----------|
| TS-001 | Parse schema with `bootstrapFlavor` specified and verify it is mapped to internal Go struct. | Unit | P1 | Yes |
| TS-002 | Verify fallback logic resolves correctly when `bootstrapFlavor` is omitted but control plane config is present. | Unit | P1 | Yes |
| TS-003 | Mock OpenStack Nova API response to return a matched flavor and verify validation passes. | Integration | P1 | Yes |
| TS-004 | Mock OpenStack Nova API response with a missing flavor and verify validation fails with expected Go field.ErrorList format. | Integration | P1 | Yes |
| TS-005 | Verify warning is printed when validation catches a flavor with only 2 vCPUs and 8 GB RAM. | Integration | P2 | Yes |
| TS-006 | Deploy a mock cluster locally via dry-run mode and verify that the generated Terraform variables.json contains the resolved flavor under the `openstack_bootstrap_flavor` key. | Integration | P1 | Yes |