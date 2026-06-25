# Product Requirements Document

**Document Version**: 1.0
**Date**: 2026-06-25
**Status**: Draft
**Ticket**: AISOS-1941

---

## 1. Executive Summary

This Product Requirements Document (PRD) defines the requirements for adding an OpenStack Designate DNS Recordset controller to the OpenStack Resource Controller (`openstack-resource-controller` or ORC). The Recordset controller will allow Kubernetes platform users to manage DNS recordsets natively using Custom Resource Definitions (CRDs). This feature will integrate directly with the existing `DNSZone` controller, enabling seamless, automated DNS record provisioning associated with managed domains.

---

## 2. Problem Statement

### 2.1 Current State
The OpenStack Resource Controller (ORC) currently supports provisioning various infrastructure components, including OpenStack Designate DNS Zones via the `DNSZone` controller. However, there is no native way to manage individual DNS records (A, AAAA, CNAME, TXT, etc.) within those zones. Platform engineers and application developers must manually configure recordsets via the OpenStack CLI/API or external DNS systems, defeating the purpose of a fully automated, Kubernetes-native infrastructure control plane.

### 2.2 Desired State
A Kubernetes-native `DNSRecordSet` controller is available in ORC. Users can declare their desired DNS recordsets in YAML format alongside their applications. The `DNSRecordSet` resource automatically references and resolves its parent `DNSZone` managed by ORC, dynamically retrieving the required OpenStack Zone ID, and manages the lifecycle (create, update, delete) of the corresponding OpenStack Designate recordset.

### 2.3 Business Impact
- **Automation & Velocity**: Eliminates manual steps or external tools for DNS management, reducing the time to provision complete application environments from hours to minutes.
- **Consistency & GitOps**: Enables complete "Infrastructure as Code" (IaC) and GitOps pipelines where DNS zone records are committed, tracked, and synchronized alongside core application workloads.
- **Error Reduction**: Reduces configuration drift and human errors (such as typos or orphaned records) by programmatically linking recordsets to their parent zones.

---

## 3. Goals & Objectives

### Primary Goals
- [ ] Implement a `DNSRecordSet` Custom Resource Definition (CRD) and controller in ORC to support full lifecycle management of OpenStack Designate recordsets.
- [ ] Provide tight integration with the existing `DNSZone` controller to automatically resolve parent zone IDs.
- [ ] Support major DNS record types (A, AAAA, CNAME, MX, TXT, SRV, NS, CAA, etc.) with standardized specification mappings.

### Success Metrics
| Metric | Current | Target | Measurement Method |
|--------|---------|--------|-------------------|
| DNS Record Management Latency | Manual API/CLI call (5-10 mins) | Under 30 seconds from CR creation to Designate reconciliation | Controller reconciler performance metrics / integration test assertions |
| Configuration Sync Accuracy | Occasional human error / drift | 0% manual drift (100% reconciliation accuracy) | Audit logs matching ORC CR status against Designate API state |
| Orphaned DNS Record Cleanup | Leftover records require manual cleanup | 100% automatic deletion on CR deletion | Integration tests tracking deletion cascades |

---

## 4. User Personas

### Persona 1: Devon (Platform Engineer)
- **Role**: Platform Engineer / DevOps Architect
- **Goals**: Provide an automated, self-service Kubernetes infrastructure for internal developers. Wants to ensure network and DNS configurations are provisioned safely, efficiently, and without manual intervention.
- **Pain Points**: Spends too much time handling JIRA tickets to add or update DNS records for application routing. Worries about orphaned records left behind when application environments are deleted.
- **Usage Context**: Devon sets up the base `DNSZone` resources and access credentials, and configures GitOps patterns so developers can safely define their own recordsets within pre-approved zones.

### Persona 2: Alex (Application Developer)
- **Role**: Cloud-Native Application Developer
- **Goals**: Deploy applications rapidly with external routing, SSL/TLS certificates, and corresponding DNS records.
- **Pain Points**: Frustrated by waiting for infrastructure teams to update DNS servers or having to switch context to OpenStack portals to create CNAMEs or TXT records.
- **Usage Context**: Alex defines a `DNSRecordSet` resource in the application's Helm chart, referencing Devon's managed `DNSZone` to dynamically register the application's hostname.

---

## 5. Requirements

### 5.1 Functional Requirements

| ID | Requirement | Priority | Acceptance Criteria |
|----|-------------|----------|---------------------|
| FR-001 | **Lifecycle CRUD Management** | MVP | The controller must support Creation, Read/Update, and Deletion of DNS recordsets in OpenStack Designate. |
| FR-002 | **Zone Integration & Reference** | MVP | The `DNSRecordSet` spec must accept a reference to an ORC `DNSZone` resource (e.g. `dnsZoneRef`). The controller must block reconciliation until the referenced `DNSZone` is Ready and has a valid zone ID in its status. |
| FR-003 | **Lifecycle Synchronization** | MVP | Deleting the `DNSRecordSet` custom resource must trigger a clean deletion of the corresponding recordset in OpenStack Designate. |
| FR-004 | **Standard DNS Record Types** | MVP | The resource spec must support standard DNS types: A, AAAA, CNAME, MX, TXT, SRV, NS, and CAA. |
| FR-005 | **Validation & Status Reporting** | MVP | The controller must validate parameters (e.g., TTL >= 0, valid IPs for A records, trailing dots for CNAMEs) and update the CR status with standard Kubernetes conditions reflecting the sync state and OpenStack resource ID. |
| FR-006 | **Cloud Credentials Reference** | MVP | The CRD must share the standard ORC `cloudCredentialsRef` pattern to locate OpenStack clouds.yaml secrets. |
| FR-007 | **Management Policies** | MVP | Support standard ORC management policies (`managed` where ORC controls the remote resource lifecycle, and `unmanaged` where ORC only links to an existing resource). |

### 5.2 Non-Functional Requirements

| ID | Requirement | Category | Target |
|----|-------------|----------|--------|
| NFR-001 | **Reconciliation Speed** | Performance | Reconcile change loop within 5 seconds of a CR state change or OpenStack event detection. |
| NFR-002 | **API Rate-Limiting & Backoff** | Reliability | Implement exponential backoff for Designate API rate-limits or temporary connection failures to avoid hammering OpenStack services. |
| NFR-003 | **Security Access Control** | Security | The controller must operate within the Kubernetes RBAC constraints, utilizing the credential secret specified in `cloudCredentialsRef` to interact with Designate. |

---

## 6. User Stories

**US-001**: As a Devon (Platform Engineer), I want to link recordsets directly to managed DNSZones so that when a zone is deleted or modified, corresponding records are safely tracked and lifecycle-managed without orphans.
- **Acceptance Criteria**:
  - Given a `DNSRecordSet` CR referencing an active `DNSZone` CR, when the `DNSZone` is deleted, the `DNSRecordSet` controller must either block deletion via finalizers or cleanly propagate finalization.
  - Given a `DNSRecordSet` CR, when the referenced `DNSZone` has not yet completed its initial creation or is in an error state, the `DNSRecordSet` controller must set a waiting status condition and avoid calling the OpenStack API with empty zone IDs.

**US-002**: As an Alex (Application Developer), I want to declare a CNAME or A record in a Kubernetes manifest so that my newly deployed microservice is immediately routable.
- **Acceptance Criteria**:
  - Given a standard application deployment manifest, when Alex applies a `DNSRecordSet` YAML specifying `type: CNAME`, `ttl: 300`, and `records: ["lb.prod.internal."]`, then ORC must create the matching CNAME record in Designate.
  - Given a synchronized `DNSRecordSet`, when Alex changes the target IP or TTL in the spec, then ORC must patch the Designate recordset in OpenStack within 10 seconds of API acceptance.

---

## 7. Scope

### In Scope
- Designing and implementing the `DNSRecordSet` CRD and associated controller in the `openstack-resource-controller` codebase.
- Integration mapping to translate the `dnsZoneRef` to OpenStack Designate Zone ID parameters.
- Standard DNS records support (A, AAAA, CNAME, MX, TXT, SRV, NS, CAA).
- Automatic reconciliation of spec updates (e.g. updating TTL or record arrays) and status propagation.
- Graceful deletion handling with finalizers.

### Out of Scope
- Automatic creation of target load balancers or ingresses (this controller only manages the recordset in Designate; integration with other services like Octavia is handled by their respective controllers).
- Multi-provider DNS synchronization (ORC interacts solely with OpenStack Designate).

---

## 8. Assumptions & Constraints

### Assumptions
- The target OpenStack environment has the Designate (DNS) service active and configured correctly.
- The existing ORC `DNSZone` controller is functioning, stable, and populates the Designate Zone UUID in its custom resource status.
- Kubernetes clusters running ORC have access and network routing to the OpenStack Designate API endpoints.

### Constraints
- The controller must match the established architectural patterns of ORC (implemented in Go, using controller-runtime, and sharing cloud client credentials logic).
- DNS validation parameters must match standard RFC and OpenStack Designate constraints (e.g., maximum record limits, trailing dot validation for records).

### Dependencies
- **Upstream OpenStack APIs**: Depend on the OpenStack Designate v2 API for recordset operations.
- **ORC Framework**: Relies on the shared credential resolution packages and the `DNSZone` v1alpha1 API schema of the same controller manager.

---

## 9. Risks & Mitigations

| Risk | Context | Likelihood | Impact | Mitigation |
|------|---------|------------|--------|------------|
| **Referential Integrity Failures** | A user creates a `DNSRecordSet` pointing to a non-existent or deleted `DNSZone` custom resource. | Medium | Medium | Implement strict cross-resource validation. Use a standard Kubernetes wait condition to block OpenStack calls, keeping the CR state in `WaitingForParentZone` until the referent is found and is Ready. |
| **OpenStack API Quota/Rate Limiting** | Rapid creation/updates of numerous DNS recordsets (e.g., during large scaling events) exceed Designate limits. | Low | High | Configure the controller's reconcile queue with rate-limiting and progressive exponential backoff. Add observability metrics for Designate API latency and error rates. |
| **Record Domain Mismatches** | The sub-domain or record name specified in the `DNSRecordSet` CR does not belong to or align with the domain name of the referenced parent `DNSZone`. | Medium | Low | Add string suffix validation inside the controller's admission path or reconciling loop to verify the record name matches the parent zone suffix before issuing the Designate API call. |

---

## 10. Timeline & Milestones

| Phase | Milestone | Target Date | Dependencies |
|-------|-----------|-------------|--------------|
| Planning: PRD | PRD approved | 2026-06-30 | Stakeholder review |
| Planning: Spec | Technical spec approved | 2026-07-05 | PRD approval |
| Planning: Epics | Epic plan approved | 2026-07-10 | Spec approval |
| Planning: Tasks | Task breakdown approved | 2026-07-15 | Epic plan approval |
| Implementation | PRs merged (all CI, AI review, and human review complete) | 2026-08-15 | Task approval |
| Documentation | Docs updated | 2026-08-20 | Implementation complete |
| Testing | QA sign-off | 2026-08-25 | Implementation complete |