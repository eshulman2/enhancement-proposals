# Product Requirements Document

**Document Version**: 1.0  
**Date**: 2026-06-17  
**Status**: Draft  
**Ticket**: AISOS-915

---

## 1. Executive Summary

This feature adds a new configuration option to the OpenShift installer for OpenStack deployments that allows users to specify a separate flavor for the bootstrap machine. Currently, the bootstrap machine inherits its flavor from the control plane configuration, forcing users to provision bootstrap nodes with the same (often expensive) resources as master nodes despite the bootstrap machine being temporary.

---

## 2. Problem Statement

### 2.1 Current State

The OpenShift installer on OpenStack currently requires the bootstrap machine to use the same flavor as control plane (master) nodes. This is explicitly documented behavior: "The bootstrap node follows the `type`, `rootVolume`, `additionalNetworkIDs`, and `additionalSecurityGroupIDs` parameters from the `controlPlane` machine pool."

In both IPI (Installer-Provisioned Infrastructure) and UPI (User-Provisioned Infrastructure) deployments, there is no mechanism to specify a different flavor for the bootstrap machine.

### 2.2 Desired State

Users can specify an optional, separate flavor for the bootstrap machine in the `install-config.yaml`. When specified, the bootstrap machine uses this flavor instead of inheriting from the control plane configuration. When not specified, the existing behavior (inheriting the control plane flavor) is preserved for backward compatibility.

### 2.3 Business Impact

- **Cost optimization**: The bootstrap machine is temporary (deprovisioned after cluster initialization). Users can now use a smaller, less expensive flavor for this short-lived node
- **Resource efficiency**: Organizations with constrained OpenStack quotas can allocate premium flavors only to persistent master nodes
- **Operational flexibility**: Different environments may have different flavor availability; decoupling bootstrap from masters removes an artificial constraint

---

## 3. Goals & Objectives

### Primary Goals

- [ ] Allow users to configure a separate OpenStack flavor for the bootstrap machine via `install-config.yaml`
- [ ] Maintain full backward compatibility when the new configuration option is not specified

---

## 4. User Personas

### Persona 1: OpenStack Platform Engineer

- **Role**: Infrastructure engineer deploying OpenShift clusters on OpenStack
- **Goals**: Deploy clusters efficiently while minimizing resource costs and staying within quota limits
- **Pain Points**: Currently forced to allocate expensive control plane flavors to the temporary bootstrap node, consuming unnecessary quota and cost
- **Usage Context**: Creating and managing OpenShift clusters on enterprise or cloud-provider OpenStack environments

---

## 5. Requirements

### 5.1 Functional Requirements

| ID | Requirement | Priority | Acceptance Criteria |
|----|-------------|----------|---------------------|
| FR-001 | The installer shall accept an optional bootstrap flavor configuration in the OpenStack platform section of `install-config.yaml` | MVP | A new field (e.g., `bootstrapFlavor` or within a bootstrap-specific machine pool) is recognized and validated by the installer |
| FR-002 | When a bootstrap flavor is specified, the bootstrap machine shall be created using that flavor | MVP | The bootstrap Nova instance is created with the user-specified flavor rather than the control plane flavor |
| FR-003 | When no bootstrap flavor is specified, the bootstrap machine shall use the control plane flavor | MVP | Existing behavior is preserved; bootstrap machine inherits the `controlPlane.platform.openstack.type` or `defaultMachinePlatform.type` value |
| FR-004 | The installer shall validate that the specified bootstrap flavor exists in the target OpenStack cloud | MVP | Installation fails with a clear error message if the specified bootstrap flavor does not exist |

### 5.2 Non-Functional Requirements

| ID | Requirement | Category | Target |
|----|-------------|----------|--------|
| NFR-001 | The feature shall not introduce breaking changes to existing install-config.yaml files | Compatibility | 100% backward compatibility with existing configurations |
| NFR-002 | Documentation shall be updated to describe the new configuration option | Documentation | Updated customization.md and relevant docs |

---

## 6. User Stories

**US-001**: As an OpenStack platform engineer, I want to specify a smaller flavor for the bootstrap machine so that I can reduce resource consumption and costs during cluster installation.

- **Acceptance Criteria**:
  - Given an `install-config.yaml` with a bootstrap flavor specified, when I run the installer, then the bootstrap machine is created with the specified flavor
  - Given an `install-config.yaml` without a bootstrap flavor specified, when I run the installer, then the bootstrap machine is created with the control plane flavor (existing behavior)
  - Given an `install-config.yaml` with an invalid bootstrap flavor name, when I run the installer, then the installation fails with a clear error indicating the flavor was not found

---

## 7. Scope

### In Scope

- New configuration field for bootstrap machine flavor in the OpenStack platform section of `install-config.yaml`
- Validation of the specified bootstrap flavor against the OpenStack cloud
- Documentation updates for the new configuration option
- Support in IPI (Installer-Provisioned Infrastructure) deployments

### Out of Scope

- Bootstrap-specific configuration for other parameters (rootVolume, securityGroups, etc.) — this feature is limited to flavor only
- Changes to UPI playbooks (users can already customize UPI deployments manually)
- Bootstrap flavor configuration for platforms other than OpenStack

---

## 8. Assumptions & Constraints

### Assumptions

- The specified bootstrap flavor meets the minimum hardware requirements for the bootstrap node (recommended: 16 GB RAM, 4 vCPUs, 100 GB disk)
- Users understand that specifying an undersized flavor may cause bootstrap failures

### Constraints

- Must not break existing install-config.yaml files that do not specify a bootstrap flavor

### Dependencies

- OpenStack Cluster API Provider (CAPO) infrastructure machine generation

---

## 9. Risks & Mitigations

| Risk | Context | Likelihood | Impact | Mitigation |
|------|---------|------------|--------|------------|
| User specifies undersized bootstrap flavor causing installation failure | Users may choose a too-small flavor to save costs, leading to bootstrap timeouts or OOM errors | Medium | Medium | Document minimum bootstrap node requirements clearly; consider adding validation warnings for flavors below recommended specs |
| Configuration naming conflicts with future features | If a full bootstrap machine pool is added later, the single-field approach may conflict | Low | Medium | Design the field placement and naming to align with potential future bootstrap machine pool configuration |

---

## 10. Timeline & Milestones

| Phase | Milestone | Target Date | Dependencies |
|-------|-----------|-------------|--------------|
| Planning: PRD | PRD approved | TBD | Stakeholder review |
| Planning: Spec | Technical spec approved | TBD | PRD approval |
| Implementation | PRs merged | TBD | Spec approval |
| Documentation | Docs updated | TBD | Implementation complete |

---

## Appendix

### A. Glossary

- **Bootstrap machine**: A temporary node responsible for initializing the control plane during cluster installation; deprovisioned after the cluster is operational
- **Flavor**: An OpenStack Nova instance type defining CPU, memory, and disk resources
- **IPI**: Installer-Provisioned Infrastructure — the installer manages all infrastructure resources
- **UPI**: User-Provisioned Infrastructure — the user manually provisions infrastructure resources

### B. References

- [OpenStack Platform Customization Documentation](https://github.com/openshift/installer/blob/main/docs/user/openstack/customization.md)
- [OpenStack MachinePool Configuration](https://github.com/openshift/installer/blob/main/pkg/types/openstack/machinepool.go)