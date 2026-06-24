# Product Requirements Document

**Document Version**: 1.0  
**Date**: 2026-06-17  
**Status**: Draft  
**Ticket**: AISOS-913

---

## 1. Executive Summary

This feature adds a dedicated configuration option for the bootstrap machine flavor in the OpenShift installer's OpenStack platform support. Currently, the bootstrap machine inherits its flavor from the control plane configuration, which forces users to over-provision the bootstrap node when their control plane requires specific high-resource flavors. This enhancement enables cost optimization and resource flexibility during cluster installation.

---

## 2. Problem Statement

### 2.1 Current State

When deploying OpenShift clusters on OpenStack, the bootstrap machine automatically uses the same Nova flavor as the control plane (master) nodes. This is hardcoded in the installer—users have no mechanism to specify a different flavor for the ephemeral bootstrap machine. The bootstrap node's flavor is derived from `controlPlane.platform.openstack.type` in the install-config.yaml.

### 2.2 Desired State

Users can independently configure the bootstrap machine's flavor through a new optional field in the OpenStack platform configuration. When specified, the bootstrap machine uses this dedicated flavor; when omitted, the current behavior (inheriting from control plane) is preserved for backward compatibility.

### 2.3 Business Impact

- **Cost reduction**: Bootstrap machines are temporary (destroyed after control plane is established) but may currently consume expensive high-resource flavors designed for production master nodes
- **Resource efficiency**: Organizations with constrained OpenStack quotas can allocate appropriate resources without over-provisioning
- **Deployment flexibility**: Enables bootstrap on environments where the control plane flavor may not be available or suitable for bootstrap operations

---

## 3. Goals & Objectives

### Primary Goals

- [ ] Enable independent flavor configuration for the bootstrap machine on OpenStack
- [ ] Maintain full backward compatibility with existing install-config.yaml files

### Success Metrics

| Metric | Current | Target | Measurement Method |
|--------|---------|--------|-------------------|
| Bootstrap flavor configurability | Not configurable | Fully configurable | Feature implementation complete |

---

## 4. User Personas

### Persona 1: OpenStack Platform Engineer

- **Role**: Infrastructure engineer deploying OpenShift clusters on private OpenStack clouds
- **Goals**: Minimize resource consumption and costs while deploying production-ready clusters
- **Pain Points**: Forced to allocate expensive control-plane-grade resources for a temporary bootstrap machine that only runs for ~30 minutes during installation
- **Usage Context**: Running `openshift-install create cluster` against OpenStack environments with specific flavor offerings and quota constraints

---

## 5. Requirements

### 5.1 Functional Requirements

| ID | Requirement | Priority | Acceptance Criteria |
|----|-------------|----------|---------------------|
| FR-001 | The installer shall accept an optional bootstrap flavor field in the OpenStack platform configuration | MVP | A new field `bootstrapFlavor` (or similar) is recognized in `platform.openstack` section of install-config.yaml |
| FR-002 | When bootstrap flavor is specified, the bootstrap machine shall be created using that flavor | MVP | Deploying with `bootstrapFlavor: m1.medium` results in a bootstrap Nova instance using the m1.medium flavor |
| FR-003 | When bootstrap flavor is not specified, the bootstrap machine shall use the control plane flavor | MVP | Existing install-config.yaml files without the new field behave identically to current behavior |
| FR-004 | The installer shall validate that the specified bootstrap flavor exists in the target OpenStack cloud | MVP | If user specifies a non-existent flavor, installer fails with a clear error message during validation |

### 5.2 Non-Functional Requirements

| ID | Requirement | Category | Target |
|----|-------------|----------|--------|
| NFR-001 | The feature shall not increase installation time | Performance | No measurable increase in install duration |
| NFR-002 | Configuration syntax shall be consistent with existing OpenStack platform options | Usability | Field naming and structure follows existing patterns |

---

## 6. User Stories

**US-001**: As an OpenStack platform engineer, I want to specify a smaller flavor for the bootstrap machine so that I can reduce resource consumption during cluster installation.

- **Acceptance Criteria**:
  - Given an install-config.yaml with `platform.openstack.bootstrapFlavor` set to "m1.small"
  - When I run `openshift-install create cluster`
  - Then the bootstrap machine is created with the m1.small flavor and control plane machines use their separately configured flavor

**US-002**: As an OpenStack platform engineer, I want the installer to default to the control plane flavor when no bootstrap flavor is specified so that existing configurations continue to work.

- **Acceptance Criteria**:
  - Given an install-config.yaml without the `bootstrapFlavor` field
  - When I run `openshift-install create cluster`
  - Then the bootstrap machine uses the same flavor as the control plane machines

---

## 7. Scope

### In Scope

- New optional `bootstrapFlavor` configuration field in the OpenStack platform section
- Validation of the bootstrap flavor against the OpenStack cloud
- Documentation updates for the new configuration option
- Backward-compatible default behavior

### Out of Scope

- Bootstrap flavor configuration for other cloud platforms (AWS, Azure, GCP, etc.)
- Changes to other bootstrap machine properties (rootVolume, networks, security groups) which already follow control plane configuration
- Changes to UPI (User-Provisioned Infrastructure) workflow—UPI already allows custom bootstrap configuration through Ansible variables

---

## 8. Assumptions & Constraints

### Assumptions

- The specified bootstrap flavor has sufficient resources to run the bootstrap process (minimum ~16GB RAM, 4 vCPUs, 100GB disk as per existing documentation)
- Users choosing a smaller flavor understand the resource requirements for successful bootstrap

### Constraints

- Must maintain API compatibility with existing install-config.yaml schema version v1
- Must integrate with the existing OpenStack validation framework

### Dependencies

- OpenStack Nova API for flavor validation
- Existing OpenStack machine pool infrastructure in the installer

---

## 9. Risks & Mitigations

| Risk | Context | Likelihood | Impact | Mitigation |
|------|---------|------------|--------|------------|
| User specifies insufficient flavor for bootstrap | Users unfamiliar with bootstrap requirements may choose flavors with inadequate resources, causing installation failures | Medium | Medium | Validate minimum resource requirements during install-config validation; provide clear error messages and documentation of minimum requirements |
| Flavor validation fails in disconnected environments | Pre-flight validation requires OpenStack API access which may be limited in air-gapped deployments | Low | Low | Handle validation failures gracefully with warnings rather than blocking errors when API is unreachable |

---

## 10. Timeline & Milestones

Specific dates are TBD and will be determined during sprint planning.

| Phase | Milestone | Target Date | Dependencies |
|-------|-----------|-------------|--------------|
| Planning: PRD | PRD approved | TBD | Stakeholder review |
| Planning: Spec | Technical spec approved | TBD | PRD approval |
| Implementation | PRs merged | TBD | Spec approval |
| Documentation | Docs updated | TBD | Implementation complete |

---

## Appendix

### A. Glossary

- **Bootstrap Machine**: A temporary OpenShift node that initializes the control plane during cluster installation. It is automatically removed once the control plane is operational.
- **Nova Flavor**: An OpenStack compute instance type defining virtual hardware specifications (vCPUs, RAM, disk).

### B. References

- [OpenShift Installer OpenStack Customization Documentation](https://github.com/openshift/installer/blob/main/docs/user/openstack/customization.md)
- [OpenStack Platform Types - machinepool.go](https://github.com/openshift/installer/blob/main/pkg/types/openstack/machinepool.go)