# Product Requirements Document

**Document Version**: 1.0
**Date**: 2026-06-24
**Status**: Draft
**Ticket**: AISOS-1936

---

## 1. Executive Summary

This feature introduces a dedicated configuration option in the OpenShift installer for OpenStack platforms to specify a distinct virtual machine flavor for the temporary bootstrap node. Currently, the bootstrap machine defaults to using the same flavor as the control plane (master) nodes. By enabling independent configuration of the bootstrap flavor, cloud administrators can optimize resource usage and reduce infrastructure costs during cluster installation without compromising the performance or size of the permanent control plane nodes.

---

## 2. Problem Statement

### 2.1 Current State
In the current implementation of `openshift-installer` for OpenStack, there is no way to decouple the virtual machine flavor of the bootstrap node from that of the master nodes. The bootstrap machine automatically inherits its flavor from the `controlPlane` platform configuration. Because the control plane nodes require high-performance, resource-heavy flavors for long-term production workloads, the temporary bootstrap machine (which only runs for a short period during initialization) is also forced to use these large, expensive flavors, leading to unnecessary resource consumption and higher infrastructure costs.

### 2.2 Desired State
The installer should provide an optional configuration parameter within the `install-config.yaml` under the OpenStack platform settings that allows administrators to explicitly specify an OpenStack flavor for the bootstrap node. If this parameter is omitted, the installer must seamlessly fall back to using the control plane flavor to ensure backward compatibility.

### 2.3 Business Impact
- **Cost Optimization**: Organizations can use smaller, cheaper instances for the short-lived bootstrap machine, resulting in direct cloud infrastructure cost savings, especially in environments with frequent cluster deployments.
- **Resource Allocation Efficiency**: In resource-constrained private or public OpenStack clouds, avoiding the allocation of a large master-sized VM for bootstrapping reduces the risk of deployment failures due to quota limits or resource exhaustion.

---

## 3. Goals & Objectives

### Primary Goals
- [ ] Provide an explicit, optional configuration setting in `install-config.yaml` to set the OpenStack flavor of the bootstrap machine.
- [ ] Maintain 100% backward compatibility by defaulting to the control plane flavor if the bootstrap-specific flavor is not configured.
- [ ] Ensure that the custom bootstrap flavor is validated against the target OpenStack cloud's available flavors before deployment starts.

### Success Metrics
| Metric | Current | Target | Measurement Method |
|--------|---------|--------|-------------------|
| Resource utilization overhead during bootstrap | Tied to Master flavor size (e.g., forced `m1.xlarge` or larger) | Customizable to smaller flavors (e.g., `m1.medium`) | Comparing OpenStack tenant hypervisor active allocations during bootstrap vs master node profiles. |
| Configuration flexibility | 0 (No ability to separate bootstrap from master flavor) | 1 (Ability to specify distinct bootstrap flavor) | Verification of schema validation and successful installation logs using separate flavors. |

---

## 4. User Personas

### Persona 1: Infrastructure Engineer / OpenStack Cloud Administrator
- **Role**: DevOps Engineer / Systems Administrator responsible for deploying and managing OpenShift clusters on OpenStack infrastructure.
- **Goals**: Successfully deploy highly-available OpenShift clusters while adhering to corporate budget limits and cloud resource quotas.
- **Pain Points**: Running out of OpenStack resource quota or incurring high costs because short-lived bootstrap nodes must be provisioned with the same high-resource flavors required by the production control plane nodes.
- **Usage Context**: Defining cluster deployment topologies within `install-config.yaml` prior to initiating the `openshift-install create cluster` command.

---

## 5. Requirements

### 5.1 Functional Requirements

| ID | Requirement | Priority | Acceptance Criteria |
|----|-------------|----------|---------------------|
| FR-001 | **Custom Bootstrap Flavor Schema**<br>The installer must expose an optional configuration parameter for specifying the OpenStack flavor specifically for the bootstrap node within `install-config.yaml` under the OpenStack platform-specific schema. | MVP | - Config validation passes when the parameter is present with a non-empty string value.<br>- Config validation passes when the parameter is absent. |
| FR-002 | **Backward Compatible Defaulting**<br>If the bootstrap flavor is not explicitly configured, the installer must fall back to using the flavor specified in the control plane platform configuration. | MVP | - Given `install-config.yaml` without the custom bootstrap flavor, when the installer executes, then the bootstrap node is provisioned with the same OpenStack flavor as the control plane nodes. |
| FR-003 | **Custom Flavor Provisioning**<br>When the bootstrap flavor is explicitly configured, the installer must provision the bootstrap node using that specific flavor. | MVP | - Given `install-config.yaml` with the custom bootstrap flavor set to `m1.medium`, when the installer provisions the bootstrap node, then the OpenStack instance for the bootstrap node is created with the `m1.medium` flavor while master nodes use their own configured flavor. |
| FR-004 | **Active Cloud Flavor Validation**<br>The installer must validate the custom bootstrap flavor against the targeted OpenStack cloud's available flavors during the pre-flight verification stage. | MVP | - Given a custom bootstrap flavor that does not exist in the target OpenStack environment, when pre-flight validation is run, then the installer must fail immediately and output a descriptive error message indicating that the specified flavor is invalid or unavailable. |

### 5.2 Non-Functional Requirements

| ID | Requirement | Category | Target |
|----|-------------|----------|--------|
| NFR-001 | **Schema Compatibility** | Usability | The new configuration field must align with existing `openshift-installer` schema conventions for other platform providers (such as AWS, Azure, GCP) that allow separate bootstrap node configuration. |
| NFR-002 | **Diagnostic Traceability** | Usability | The installer's standard logging must clearly output the selected flavor used for the bootstrap node (whether default or custom) during the node provisioning step. |

---

## 6. User Stories

**US-001**: As an Infrastructure Engineer, I want to configure a separate OpenStack flavor for the bootstrap node in `install-config.yaml` so that I can deploy OpenShift clusters without exceeding tight resource quotas on temporary infrastructure.
- **Acceptance Criteria**:
  - Given a valid `install-config.yaml` containing the custom bootstrap flavor configuration, when I run `openshift-install create cluster`, then the pre-flight validation succeeds.
  - Given that the installation has started, when the bootstrap VM is provisioned in OpenStack, then its instance type corresponds to the configured bootstrap flavor.
  - Given that the bootstrap VM is running, when master nodes are provisioned, then their instance types correspond to the larger control plane flavor, confirming the flavors are decoupled.

---

## 7. Scope

### In Scope
- Addition of an optional configuration parameter for the OpenStack bootstrap flavor within the installer's schema.
- Fallback logic to inherit the control plane flavor when the parameter is omitted.
- Pre-flight validation of the bootstrap-specific flavor against the target OpenStack API.
- Integration with provisioning logic used by the OpenStack provider in the installer to apply the distinct flavor.

### Out of Scope
- Support for modifying the bootstrap machine's flavor post-deployment (the bootstrap machine is temporary and destroyed after installation).
- Decoupling of other bootstrap-specific resources on OpenStack (such as separate bootstrap networks, security groups, or storage types) unless already supported or out of scope for this specific config.
- Decoupling of bootstrap flavors for other cloud providers (this is specifically for OpenStack).

---

## 8. Assumptions & Constraints

### Assumptions
- The targeted OpenStack cloud is accessible and active during the installation process to allow API query validation of available flavors.
- The user has appropriate quotas in OpenStack for both the custom bootstrap flavor and the control plane node flavors.

### Constraints
- The schema design must conform to the upstream `openshift-installer` structural guidelines for configuration files.
- The bootstrap flavor must satisfy the minimum OS and OpenShift requirements (such as minimum RAM and vCPU requirements for the bootstrap node to execute successfully).

### Dependencies
- OpenStack API availability for pre-flight flavor verification.
- Upstream `openshift-installer` dependency on the OpenStack provisioning clients and Go SDK wrappers used to provision instances.

---

## 9. Risks & Mitigations

| Risk | Mitigation Strategy |
|------|---------------------|
| User configures a custom bootstrap flavor that is too small (e.g., insufficient RAM/CPU), causing the bootstrap process to fail or hang. | Define minimum resource recommendations in the documentation, and optionally implement a pre-flight warning if the selected flavor does not meet the recommended minimum specs. |
| Configuration drift or schema breakage across installer versions if the parameter name is inconsistent with other platform schemas. | Design the field naming and structure to align with existing patterns in other platforms (such as GCP, AWS, Azure) that support bootstrap-specific instance type overrides. |