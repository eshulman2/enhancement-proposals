# Product Requirements Document

**Document Version**: 1.0  
**Date**: 2026-06-17  
**Status**: Draft  
**Ticket**: AISOS-900

---

## 1. Executive Summary

This feature adds a dedicated configuration option to specify an independent OpenStack flavor for the bootstrap machine in the OpenShift installer. Currently, the bootstrap machine inherits its flavor from the control plane configuration, which forces users to provision expensive control plane-sized instances for a temporary node that is deleted after cluster initialization.

---

## 2. Problem Statement

### 2.1 Current State

When deploying OpenShift on OpenStack, the bootstrap machine automatically inherits the `type` (flavor) configuration from the `controlPlane` machine pool. The bootstrap node follows the `type`, `rootVolume`, `additionalNetworkIDs`, and `additionalSecurityGroupIDs` parameters from the `controlPlane` machine pool without any option for independent configuration. This coupling is documented in the OpenShift installer's OpenStack customization documentation.

### 2.2 Desired State

Users can specify a separate OpenStack flavor for the bootstrap machine independent of the control plane flavor, allowing them to right-size the temporary bootstrap instance based on its actual resource requirements rather than the requirements of permanent control plane nodes.

### 2.3 Business Impact

- **Cost reduction**: Bootstrap machines exist only during cluster installation (typically 15-30 minutes). Using smaller flavors reduces OpenStack compute costs during cluster provisioning.
- **Resource efficiency**: Organizations with limited OpenStack quotas can allocate resources more precisely without over-provisioning temporary infrastructure.
- **Flexibility**: Allows optimization for environments where control plane requirements differ significantly from bootstrap requirements.

---

## 3. Goals & Objectives

### Primary Goals

- [ ] Enable independent flavor configuration for the bootstrap machine separate from the control plane
- [ ] Maintain backward compatibility by defaulting to control plane flavor when bootstrap flavor is not specified

### Success Metrics

| Metric | Current | Target | Measurement Method |
|--------|---------|--------|-------------------|
| Bootstrap flavor configurability | Not configurable | Fully configurable | Feature availability in install-config.yaml |
| Backward compatibility | N/A | 100% existing configs work unchanged | Automated testing |

---

## 4. User Personas

### Persona 1: OpenStack Platform Engineer

- **Role**: Platform engineer responsible for deploying and managing OpenShift clusters on private OpenStack infrastructure
- **Goals**: Optimize resource utilization and reduce infrastructure costs while maintaining reliable cluster deployments
- **Pain Points**: Forced to use expensive control plane-sized flavors for temporary bootstrap nodes; limited OpenStack quotas make over-provisioning problematic
- **Usage Context**: Running multiple cluster deployments for development, testing, and production environments

---

## 5. Requirements

### 5.1 Functional Requirements

| ID | Requirement | Priority | Acceptance Criteria |
|----|-------------|----------|---------------------|
| FR-001 | Provide a configuration option to specify the bootstrap machine flavor independently from the control plane flavor | MVP | When `bootstrapFlavor` (or equivalent field) is specified in the OpenStack platform configuration, the bootstrap machine uses that flavor instead of inheriting from controlPlane |
| FR-002 | Default to control plane flavor when bootstrap flavor is not specified | MVP | Existing install-config.yaml files without the new field continue to work, with bootstrap inheriting the control plane flavor |
| FR-003 | Validate that the specified bootstrap flavor exists in OpenStack before attempting to create the bootstrap machine | MVP | Installer fails with a clear error message if the specified bootstrap flavor does not exist |

### 5.2 Non-Functional Requirements

| ID | Requirement | Category | Target |
|----|-------------|----------|--------|
| NFR-001 | Configuration field follows existing naming conventions and YAML structure | Usability | Consistent with other OpenStack platform configuration options |
| NFR-002 | Documentation updated to describe the new configuration option | Usability | Field documented in customization.md with examples |

---

## 6. User Stories

**US-001**: As an OpenStack platform engineer, I want to specify a smaller flavor for the bootstrap machine so that I can reduce resource consumption and costs during cluster installation.

- **Acceptance Criteria**:
  - Given an install-config.yaml with a `bootstrapFlavor` field in the OpenStack platform configuration, when I run `openshift-install create cluster`, then the bootstrap machine is created with the specified flavor
  - Given an install-config.yaml without a `bootstrapFlavor` field, when I run `openshift-install create cluster`, then the bootstrap machine uses the control plane flavor (existing behavior preserved)

---

## 7. Scope

### In Scope

- New configuration field for bootstrap flavor in the OpenStack platform configuration
- Validation of the bootstrap flavor against available OpenStack flavors
- Documentation updates for the new configuration option
- Backward compatibility with existing install-config.yaml files

### Out of Scope

- Separate configuration for other bootstrap machine properties (rootVolume, networks, security groups) — these continue to inherit from controlPlane
- Changes to UPI (User Provided Infrastructure) workflows
- Bootstrap flavor configuration for other cloud platforms

---

## 8. Assumptions & Constraints

### Assumptions

- The bootstrap machine's minimum resource requirements (16 GB RAM, 4 vCPUs, 100 GB disk as documented) are understood by users selecting a flavor
- Users have access to query available flavors in their OpenStack environment

### Constraints

- The configuration schema must maintain backward compatibility with existing install-config.yaml files
- The implementation must align with the existing MachinePool and Platform type structures in the installer codebase

### Dependencies

- OpenStack compute API availability for flavor validation
- Cluster API Provider OpenStack (CAPO) for bootstrap machine provisioning

---

## 9. Risks & Mitigations

| Risk | Context | Likelihood | Impact | Mitigation |
|------|---------|------------|--------|------------|
| User specifies an undersized flavor causing bootstrap failure | Users unfamiliar with bootstrap requirements may select flavors with insufficient resources, leading to failed cluster installations | Med | Med | Document minimum resource requirements alongside the new configuration option; provide clear error messages when bootstrap fails due to resource constraints |
| Configuration field naming inconsistency | The field name or location may not follow established patterns, causing user confusion | Low | Low | Review existing OpenStack platform configuration patterns and follow established naming conventions (e.g., `type` for flavor in MachinePool) |

---

## 10. Timeline & Milestones

Timeline details are TBD pending project planning.

---

## Appendix

### A. Glossary

- **Bootstrap machine**: A temporary OpenStack instance created during OpenShift installation that stands up the initial control plane. It is automatically deleted once the production control plane is operational.
- **Flavor**: An OpenStack Nova instance type that defines the compute, memory, and storage capacity of virtual machine instances.

### B. References

- [OpenShift Installer OpenStack Customization Documentation](https://github.com/openshift/installer/blob/main/docs/user/openstack/customization.md)
- [OpenStack Platform Types - machinepool.go](https://github.com/openshift/installer/blob/main/pkg/types/openstack/machinepool.go)
- [OpenStack Platform Types - platform.go](https://github.com/openshift/installer/blob/main/pkg/types/openstack/platform.go)