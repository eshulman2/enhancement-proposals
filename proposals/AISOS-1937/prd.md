# Product Requirements Document

**Document Version**: 1.0
**Date**: 2026-06-25
**Status**: Draft
**Ticket**: AISOS-1937

---

## 1. Executive Summary

This feature introduces an optional configuration parameter, `bootstrapFlavor`, within the OpenStack platform-specific configuration of the OpenShift Installer (`install-config.yaml`). This allows cluster administrators to specify a distinct OpenStack flavor for the temporary bootstrap machine, independent of the control plane flavor. By decoupling these configurations, administrators can avoid deploying the temporary bootstrap node with expensive, highly specialized control-plane-optimized hardware configurations (such as NFV or NUMA pinning), reducing resource waste and avoiding provisioning failures.

---

## 2. Problem Statement

### 2.1 Current State
In the current implementation of `openshift-installer` on OpenStack, the temporary bootstrap machine inherits its VM flavor directly from the control plane machine pool flavor shared with the master nodes. 

### 2.2 Desired State
Administrators must be able to specify a separate OpenStack flavor dedicated only to the temporary bootstrap machine using a new `bootstrapFlavor` parameter in the `install-config.yaml` file. If this parameter is omitted, the installer must fall back to using the control plane flavor, preserving backward compatibility.

### 2.3 Business Impact
This change enables successful deployments of OpenShift in enterprise OpenStack environments where master nodes require specialized, resource-intensive, or tightly constrained flavors (e.g., NUMA pinning, huge pages, or SR-IOV configurations). Administrators will no longer waste premium cloud resources or experience deployment failures on the temporary bootstrap machine due to host compatibility or quota limits.

---

## 3. Goals & Objectives

### Primary Goals
- [ ] Provide an optional configuration field (`bootstrapFlavor`) under the OpenStack platform configuration in `install-config.yaml`.
- [ ] Ensure backward compatibility is maintained by defaulting to the control plane machine pool flavor when `bootstrapFlavor` is not specified.
- [ ] Validate the existence and usability of the specified `bootstrapFlavor` on the target OpenStack cloud during pre-flight installer validation checks.

### Success Metrics
| Metric | Current | Target | Measurement Method |
|--------|---------|--------|-------------------|
| Configuration Flexibility | 0% of deployments can decouple bootstrap flavor from control plane flavor. | 100% of deployments can specify different flavors for bootstrap and master nodes. | Verification of provisioned VM metadata in OpenStack after installation. |
| Backward Compatibility | N/A | 100% of existing `install-config.yaml` templates without the field continue to deploy successfully. | CI execution of standard installations with omitted `bootstrapFlavor`. |
| Validation Accuracy | 0% of invalid bootstrap flavors are caught before VM provisioning starts. | 100% of invalid/non-existent bootstrap flavor names trigger a pre-flight validation error. | Pre-flight validation unit tests and CLI integration tests with invalid values. |

---

## 4. User Personas

### Persona 1: DevOps / Infrastructure Engineer (OpenStack Operator)
- **Role**: Cloud Infrastructure Architect & Cluster Administrator
- **Goals**: Provision and manage production-grade OpenShift clusters on OpenStack while keeping resource utilization highly optimized and ensuring smooth cluster bootstrapping.
- **Pain Points**: Master nodes in production require heavy NUMA-pinned hardware flavors, but hypervisors with this capability are scarce or tightly constrained. Bootstrapping fails or wastes expensive resources because the temporary bootstrap VM also requests the NUMA-pinned master flavor.
- **Usage Context**: Creates and executes the `install-config.yaml` file to provision clusters on OpenStack via the `openshift-install` command-line utility.

---

## 5. Requirements

### 5.1 Functional Requirements

| ID | Requirement | Priority | Acceptance Criteria |
|----|-------------|----------|---------------------|
| FR-001 | Support optional `bootstrapFlavor` string property in OpenStack platform config. | MVP | Given an `install-config.yaml` with `platform.openstack.bootstrapFlavor` specified, the configuration schema validation must parse successfully. |
| FR-002 | Provision the bootstrap VM using the specified `bootstrapFlavor`. | MVP | Given a valid `bootstrapFlavor` in the config, when running cluster provisioning, the installer must launch the OpenStack bootstrap instance using that exact flavor name. |
| FR-003 | Default to control plane flavor when `bootstrapFlavor` is omitted. | MVP | Given an `install-config.yaml` without `bootstrapFlavor`, the installer must fall back to using the flavor specified in `controlPlane.platform.openstack.type` or `platform.openstack.defaultMachinePlatform.type`. |
| FR-004 | Validate the specified `bootstrapFlavor` exists on target OpenStack cloud. | MVP | Given an invalid flavor name in `bootstrapFlavor`, when running validation checks, the installer must fail immediately with a clear error: `flavor "<invalid_flavor>" not found`. |

### 5.2 Non-Functional Requirements

| ID | Requirement | Category | Target |
|----|-------------|----------|--------|
| NFR-001 | Configuration Backward Compatibility | Compatibility | Upgrading the installer binary must not break the parsing of existing `install-config.yaml` configurations that lack the `bootstrapFlavor` property. |
| NFR-002 | Comprehensive Documentation | Usability | Clear instructions detailing `bootstrapFlavor` usage, inheritance rules, and recommended resources must be added to the OpenStack customization documentation. |

---

## 6. User Stories

**US-001**: As an OpenStack Cloud Administrator, I want to configure a standard, general-purpose flavor for the temporary bootstrap machine while assigning a highly specialized NUMA-pinned flavor for the control plane masters, so that I can conserve constrained hardware resource slots and guarantee successful bootstrap execution.
- **Acceptance Criteria**:
  - Given an OpenStack cloud where NUMA-pinned flavors are restricted to specific hypervisor hosts,
  - When I define a NUMA-pinned flavor for control plane masters and set `bootstrapFlavor: m1.large` in `install-config.yaml`,
  - Then the installer provisions the temporary bootstrap VM using `m1.large` on general-purpose compute resources, while provisioning masters using the NUMA-pinned flavor.

---

## 7. Scope

### In Scope
- Addition of the optional `bootstrapFlavor` field to the OpenStack platform-specific installer schema.
- Verification logic to fetch and validate the bootstrap flavor from the OpenStack API before provisioning resources.
- Terraform variables and deployment templates updated to map the customized bootstrap flavor to the temporary bootstrap instance.
- Updating user documentation to detail `bootstrapFlavor` configuration.

### Out of Scope
- Configuring other properties (such as root volume size, networks, or security groups) for the bootstrap node independently from the control plane inheritance model.
- Decoupling bootstrap flavors on non-OpenStack platforms under this specification.

---

## 8. Assumptions & Constraints

### Assumptions
- The target OpenStack environment has the specified flavor created and active.
- The credentials configured in `clouds.yaml` have sufficient permissions and quota to fetch the details of and spawn the specified bootstrap flavor.
- The specified flavor meets the minimum resource demands required for the OpenShift bootstrap process to execute successfully (e.g., minimum 4 vCPUs, 16 GB RAM).

### Constraints
- The installer schema must continue to parse successfully even when configuration files conform to previous installer schema versions.

### Dependencies
- OpenStack APIs (`Nova` compute service) must be available and reachable during the pre-flight check phase to run validation queries.

---

## 9. Risks & Mitigations

| Risk | Impact | Severity | Mitigation Strategy |
|------|--------|----------|---------------------|
| User configures an under-provisioned flavor (e.g., `m1.tiny` / 1 vCPU) causing bootstrap timeouts or Out-Of-Memory (OOM) failures. | Cluster installation fails during bootstrap. | Medium | Add warnings or validation checks if the selected `bootstrapFlavor` does not meet the minimum recommended resource profile (e.g., 4 vCPUs and 16 GB RAM). |
| Offline validation fails when the OpenStack compute API is temporarily unresponsive. | Installer exits before provisioning. | Low | Utilize the installer's existing robust connectivity checking and retry mechanisms during the validation phase. |