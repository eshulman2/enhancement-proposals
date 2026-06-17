# Product Requirements Document

**Document Version**: 1.0  
**Date**: 2026-06-17  
**Status**: Draft  
**Ticket**: AISOS-914

---

## 1. Executive Summary

This feature adds a dedicated configuration option for specifying the OpenStack flavor used by the bootstrap machine during OpenShift cluster installation. Currently, the bootstrap machine inherits its flavor from the control plane configuration, preventing independent sizing of this temporary installation node.

---

## 2. Problem Statement

### 2.1 Current State

When deploying OpenShift on OpenStack using the IPI (Installer Provisioned Infrastructure) method, the bootstrap machine automatically inherits its flavor configuration from the `controlPlane.platform.openstack.type` field. This means the bootstrap node—a temporary machine used only during cluster initialization—must use the same compute resources (CPU, memory, disk) as the production control plane nodes.

### 2.2 Desired State

Operators can specify a distinct OpenStack flavor for the bootstrap machine, independent of the control plane flavor. This allows right-sizing the bootstrap node based on its actual resource requirements during installation rather than production control plane requirements.

### 2.3 Business Impact

- **Cost optimization**: Bootstrap machines are temporary (destroyed after cluster initialization). Using smaller flavors for bootstrap reduces cloud resource costs during installation.
- **Resource efficiency**: Organizations with constrained OpenStack quotas can allocate control plane resources more efficiently by not over-provisioning the bootstrap node.
- **Flexibility**: Enables installation scenarios where the bootstrap node requirements differ from control plane requirements (e.g., different disk types, availability zones, or resource constraints).

---

## 3. Goals & Objectives

### Primary Goals

- [ ] Enable independent configuration of the bootstrap machine OpenStack flavor
- [ ] Maintain backward compatibility with existing install configurations

### Success Metrics

| Metric | Current | Target | Measurement Method |
|--------|---------|--------|-------------------|
| Bootstrap flavor configurability | Not available | Available via install-config.yaml | Feature deployment verification |

---

## 4. User Personas

### Persona 1: OpenStack Platform Engineer

- **Role**: Infrastructure engineer deploying OpenShift clusters on OpenStack
- **Goals**: Deploy OpenShift clusters with optimal resource utilization; minimize cloud costs during installation
- **Pain Points**: Forced to use oversized control plane flavors for the temporary bootstrap machine; cannot optimize bootstrap node resources independently
- **Usage Context**: Cluster installation and lifecycle management using openshift-install tooling

---

## 5. Requirements

### 5.1 Functional Requirements

| ID | Requirement | Priority | Acceptance Criteria |
|----|-------------|----------|---------------------|
| FR-001 | The installer shall support a new configuration field to specify the OpenStack flavor for the bootstrap machine | MVP | Given an install-config.yaml with a bootstrap flavor field specified, when the installer creates the bootstrap machine, then it uses the specified flavor instead of the control plane flavor |
| FR-002 | If the bootstrap flavor is not specified, the installer shall fall back to the current behavior (inherit from control plane) | MVP | Given an install-config.yaml without a bootstrap flavor field, when the installer creates the bootstrap machine, then it uses the control plane flavor as it does today |
| FR-003 | The installer shall validate that the specified bootstrap flavor exists in the target OpenStack cloud | MVP | Given an install-config.yaml with an invalid bootstrap flavor, when running the installer, then a clear validation error is returned before attempting installation |

### 5.2 Non-Functional Requirements

| ID | Requirement | Category | Target |
|----|-------------|----------|--------|
| NFR-001 | The new configuration field must follow existing OpenStack platform configuration patterns | Usability | Configuration style consistent with other OpenStack machine pool properties |
| NFR-002 | Backward compatibility with existing install-config.yaml files | Compatibility | All existing configurations continue to work without modification |

---

## 6. User Stories

**US-001**: As an OpenStack platform engineer, I want to specify a different OpenStack flavor for the bootstrap machine so that I can use smaller/cheaper resources for the temporary bootstrap node while maintaining larger flavors for production control plane nodes.

- **Acceptance Criteria**:
  - Given I am creating an install-config.yaml for an OpenStack deployment, when I specify a flavor under a bootstrap-specific configuration field, then the bootstrap machine is created with that flavor
  - Given I specify an invalid or non-existent flavor for the bootstrap machine, when I run the installer validation, then I receive a clear error message indicating the flavor is not available
  - Given I do not specify a bootstrap flavor, when the installer creates the bootstrap machine, then it inherits the flavor from the control plane configuration (existing behavior)

---

## 7. Scope

### In Scope

- New configuration field for bootstrap machine flavor in the OpenStack platform section
- Validation of the bootstrap flavor against the target OpenStack cloud
- Documentation updates for the new configuration option
- Backward-compatible default behavior when the field is not specified

### Out of Scope

- Bootstrap-specific configuration for other properties (rootVolume, zones, etc.) beyond flavor—these already inherit from controlPlane as documented
- Changes to bootstrap machine behavior on other platforms (AWS, GCP, Azure, etc.)
- UPI (User Provisioned Infrastructure) installation methods

---

## 8. Assumptions & Constraints

### Assumptions

- The OpenStack cloud has the specified bootstrap flavor available and accessible to the installer credentials
- Users understand that the bootstrap machine is temporary and destroyed after cluster initialization
- The minimum resource requirements for the bootstrap machine will be documented for users selecting smaller flavors

### Constraints

- Must integrate with the existing install-config.yaml schema and validation framework
- Must maintain API consistency with existing OpenStack platform configuration patterns

### Dependencies

- openshift/installer repository codebase
- Existing OpenStack platform type definitions (`pkg/types/openstack`)
- Cluster API Provider OpenStack (CAPO) for machine creation

---

## 9. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Users select flavors with insufficient resources for bootstrap, causing installation failures | Medium | Medium | Document minimum bootstrap resource requirements; consider validation warnings for undersized flavors |
| Increased configuration complexity for users who don't need this feature | Low | Low | Make the field optional with sensible defaults (inherit from control plane) |
| Schema changes break existing automation or tooling | Low | High | Ensure backward compatibility; new field is additive and optional |

---

## 10. Glossary

| Term | Definition |
|------|------------|
| Bootstrap machine | A temporary machine created during OpenShift installation that runs the initial cluster bootstrap process; destroyed after control plane initialization |
| Flavor | An OpenStack Nova compute resource template defining CPU, memory, disk, and other compute characteristics |
| IPI | Installer Provisioned Infrastructure - installation method where the OpenShift installer creates and manages cloud resources |
| Control plane | The set of master nodes running Kubernetes control plane components (API server, etcd, controllers) |