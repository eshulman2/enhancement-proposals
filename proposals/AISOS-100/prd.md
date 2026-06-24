# Product Requirements Document: Architecture Plan Generation Prompt with Constitution Compliance

**Document Version**: 1.0  
**Date**: 2026-06-24  
**Status**: Draft  
**Ticket**: AISOS-64 / AISOS-100  

---

## 1. Executive Summary

This feature introduces a structured **Architecture Plan Generation Prompt Template** within the AI-Integrated SDLC Orchestrator (Forge). The prompt template enables the AI system to generate comprehensive, high-level technical blueprints for new features prior to Epic and Task decomposition. By embedding the repository’s `constitution.md` directly into the generation prompt, the system guarantees that all automated technical designs strictly comply with project-specific architectural principles, tech stack guidelines, and quality guardrails.

---

## 2. Problem Statement

### 2.1 Current State
The SDLC Orchestrator currently transitions from behavioral specifications directly to Epic and Task decomposition without an intermediate system-level design phase. This lack of architectural planning introduces several issues:
- **Design Fragility**: Complex features are broken down without a unified blueprint (such as schemas, sequence flows, or service boundaries), leading to disjointed task execution.
- **Guardrail Violations**: The system does not systematically verify its high-level design choices against the repository's `constitution.md` during planning, occasionally leading to code implementations that violate core project constraints (e.g., using unauthorized libraries, violating data sovereignty rules, or ignoring established design patterns).
- **Review Overhead**: Human Tech Leads spend significant time during pull request reviews identifying and correcting basic architectural alignment errors.

### 2.2 Desired State
A dedicated **Architecture Planning** phase is established. The orchestrator uses a new, versioned prompt template to generate an Architecture Plan. This prompt loads the repository's `constitution.md` (or standard equivalent) and mandates that the generated design strictly adhere to its principles. The final output is a structured technical plan that includes a compliance matrix, verifying and documenting adherence to every relevant project rule.

### 2.3 Business Impact
- **Reduced Development Rework**: Decreases architectural rework and refactoring cycles during code execution and PR review phases by an estimated 40%.
- **Design Compliance**: Guarantees 100% compliance with non-negotiable repository guardrails prior to executing any code generator tasks.
- **Faster Onboarding for Subagents**: Provides downstream AI developer subagents with crystal-clear, compliant technical boundaries, resulting in higher first-pass success rates for automated task implementations.

---

## 3. Goals & Objectives

### Primary Goals
- **G-001**: Establish a standard, versioned architecture plan generation prompt template at `src/forge/prompts/v1/generate-architecture-plan.md`.
- **G-002**: Enforce 100% adherence to project-level rules by instructing the LLM to ingest and map the workspace’s `constitution.md` within the generated design.
- **G-003**: Standardize the output format of the Architecture Plan to include system boundaries, data models, API contracts, and a structured compliance verification table.

### Success Metrics
| Metric | Current | Target | Measurement Method |
|--------|---------|--------|-------------------|
| **Constitution Compliance Rate** | 0% (no plan check) | 100% | Automated check of generated plans to ensure the compliance matrix is present and contains no unmitigated violations. |
| **Technical Design Sufficiency** | N/A (no plan stage) | >= 90% | Percentage of generated plans accepted by the Tech Lead without structural changes. |
| **Planning Phase Performance** | N/A | < 5 minutes | Average time taken by the system to load context, execute the prompt, and output the plan. |

---

## 4. User Personas

### Persona 1: Tech Lead (Alex)
- **Role**: Software Architect & Technical Reviewer
- **Goals**: Maintain long-term system health, ensure all automated and manual features align with project architectural patterns, and enforce compliance with security and operational policies.
- **Pain Points**: Reviewing large, complex AI-generated PRs is slow and frustrating when basic design principles (e.g., modular boundaries, database conventions) are violated.
- **Usage Context**: Approves a feature's specification and triggers the "Planning" transition, then reviews the generated Architecture Plan for compliance before Epics and Tasks are created.

### Persona 2: AI Developer Agent (Forge-Deep-Agent)
- **Role**: Autonomous SDLC Code Implementer
- **Goals**: Correctly implement tasks, pass local unit tests, and compile successfully within container sandboxes without breaking existing functionality.
- **Pain Points**: Vague, high-level task descriptions without clear data schema definitions, library constraints, or interface requirements lead to failing test suites and rejected pull requests.
- **Usage Context**: Automatically reads the approved Architecture Plan and Epics to guide file-level edits, module creations, and schema migrations.

---

## 5. Requirements

### 5.1 Functional Requirements

| ID | Requirement | Priority | Acceptance Criteria |
|----|-------------|----------|---------------------|
| **FR-001** | **Prompt Template Creation**<br>The system MUST define a prompt template at `src/forge/prompts/v1/generate-architecture-plan.md`. | MVP | - The file is created and accessible by the prompt loading system.<br>- The template includes placeholders: `{spec_content}`, `{constitution_content}`, `{context}`, and `{project_key}`. |
| **FR-002** | **Project Constitution Ingestion**<br>The prompt MUST instruct the model to ingest, analyze, and apply the principles defined in the workspace's `constitution.md` (or standard fallback guardrails if none is found). | MVP | - The template includes instructions on how to parse the `{constitution_content}` variable.<br>- The generated plan references the exact principle names or IDs from the loaded constitution. |
| **FR-003** | **Plan Structural Integrity**<br>The prompt MUST enforce that the generated Architecture Plan is written in valid Markdown and contains the following mandatory sections:<br>1. Executive Summary<br>2. Component & System Boundaries<br>3. Data Model & Schema Changes<br>4. API Contracts & Communication Protocols<br>5. Project Constitution Compliance Matrix<br>6. Risks & Technical Trade-offs | MVP | - Output validation checks verify that all six headings exist in the generated plan.<br>- Plans missing any of these sections are marked as invalid. |
| **FR-004** | **Mermaid.js Diagram Enforcement**<br>The prompt MUST instruct the model to provide at least one sequence or flow diagram in Mermaid.js format representing the new system interactions under the Component & System Boundaries section. | MVP | - The generated plan contains a valid ```mermaid block block illustrating the components and event flow. |
| **FR-005** | **Project Constitution Compliance Matrix**<br>The prompt MUST require the output to include a Markdown table that maps each relevant principle from `constitution.md` to the design decisions made. | MVP | - The table contains columns: `Principle ID/Name`, `Status` (Compliant / Justified Deviation / N/A), and `Design Mapping / Implementation Detail`. |
| **FR-006** | **Deviation Justification**<br>If a proposed design choice conflicts with a principle in `constitution.md`, the plan MUST explicitly mark the status as `Justified Deviation` and document the specific technical reason and trade-offs. | MVP | - Conflicting decisions are flagged with a prominent warning prefix (e.g. `[DEVIATION]`) in the compliance table.<br>- A dedicated subsection under "Risks & Technical Trade-offs" details all such deviations. |
| **FR-007** | **Graceful Fallback Handling**<br>The prompt MUST define fallback behavior when `{constitution_content}` is empty, directing the model to apply industry-standard architectural best practices for the target tech stack. | MVP | - When no constitution is supplied, the prompt instructs the model to build the Compliance Matrix against default guardrails (e.g. testing coverage, security standards, modularity). |

### 5.2 Non-Functional Requirements

| ID | Requirement | Category | Target |
|----|-------------|----------|--------|
| **NFR-001** | **Structural Conformity** | Usability / Reliability | 95%+ of generation runs produce valid Markdown with correctly formatted compliance tables and Mermaid diagrams that parse without error. |
| **NFR-002** | **Code Generation Avoidance** | Security / Maintainability | The prompt must strictly forbid the inclusion of full source code implementation files, limiting output to blueprints, schema definitions, and pseudocode where necessary. |
| **NFR-003** | **Context Window Efficiency** | Performance | The prompt template must be lightweight (< 1,200 tokens) to ensure maximum available token headroom for large specifications and multi-principle constitution documents. |

---

## 6. User Stories

### US-001: Architecture Plan Generation
As a Tech Lead, I want the system to automatically generate a detailed Architecture Plan from an approved specification so that I can inspect the system design before any Epics or Tasks are created.
- **Acceptance Criteria**:
  - **Given** an approved feature specification, **When** the workflow transitions to the Architecture Planning phase, **Then** the orchestrator reads `generate-architecture-plan.md` and populates `{spec_content}` and `{constitution_content}`.
  - **Given** the compiled prompt is executed, **When** the model returns its response, **Then** the generated plan contains the six mandatory sections, including a Mermaid diagram representing component interactions.
  - **Given** the generated plan is completed, **When** it is saved to the workspace, **Then** it contains only the Markdown document with no conversational introduction or AI meta-commentary.

### US-002: Constitution-Driven Planning and Verification
As a Tech Lead, I want the generated Architecture Plan to include a verified compliance matrix matching our project's `constitution.md` rules so that I can easily identify deviations or non-compliant design decisions.
- **Acceptance Criteria**:
  - **Given** a workspace with a `constitution.md` that mandates "All persistence must use Postgres" and "API contracts must use JSON schemas", **When** the architecture plan is generated, **Then** the Project Constitution Compliance Matrix includes rows for both principles.
  - **Given** the compliance matrix, **When** a design choice violates a principle (e.g., using a Redis key-value store for primary relational data), **Then** the matrix lists the status as `Justified Deviation` and provides a clear technical rationale for the human reviewer.
  - **Given** a workspace that does *not* contain a `constitution.md` file, **When** the planner executes, **Then** the prompt engine injects default SDLC standards, logs a workspace setup warning, and generates a plan with a standard compliance matrix.

---

## 7. Scope

### In Scope
- Creating the prompt template file `src/forge/prompts/v1/generate-architecture-plan.md`.
- Defining variables to inject specification text, project constitution content, and project keys.
- Requiring specific architectural details in the generated output, such as data schemas, interface models, flow diagrams, and a structured compliance matrix.
- Defining fallback rules for repositories lacking a `constitution.md`.

### Out of Scope
- Implementing the runtime node in LangGraph or updating orchestrator transition logic (this PRD specifies only the prompt template and output requirements).
- Writing automatic programmatic linters to parse and validate Mermaid.js code syntax.
- Auto-reverting or blocking the workflow on a `Justified Deviation` (all deviations must be flagged for manual review but are allowed to generate).

---

## 8. Assumptions & Constraints

### Assumptions
- The workspace setup node will correctly parse and load the target repository's `constitution.md` or `CONSTITUTION.md` and pass it to the state as a string.
- Downstream planning steps, such as `decompose-epics.md`, will eventually be updated to receive the generated Architecture Plan alongside the spec to ensure consistency.
- The LLM used (e.g., Claude 3.5 Sonnet) is capable of following instructions to output structured, clean Markdown tables and valid Mermaid.js diagrams.

### Constraints
- The prompt template must reside under the existing versioned prompts path: `src/forge/prompts/v1/`.
- The prompt must be designed to avoid hardcoded project parameters, remaining fully generalizable across any repository that Forge orchestrates.

---

## 9. Risks & Mitigations

| Risk | Description | Mitigation Strategy |
|------|-------------|---------------------|
| **Hallucinated Compliance** | The LLM may assert full compliance with a constitution principle while presenting a design that actually violates it. | The prompt template instructs the model to provide specific, testable design mappings (e.g., listing specific file paths, database tables, or endpoints) to justify compliance rather than using vague, generic statements. |
| **Constitution Size Inflation** | Large, detailed project constitutions may consume too much of the context window or cause the model to lose track of key constraints. | The prompt template instructs the model to prioritize principles that govern data persistence, architectural boundaries, security, and interface design, while summarizing lower-priority formatting guidelines. |
| **Invalid Mermaid Syntax** | The model may produce syntactically incorrect Mermaid.js diagrams that fail to render in user interfaces. | The prompt specifies strict, simple Mermaid diagram rules (e.g., sequence diagram or flow chart formats only) and forbids complex formatting options that frequently cause parsing errors. |