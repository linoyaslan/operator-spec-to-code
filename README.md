# operator-spec-to-code
Provides four Claude agents that enable a spec-driven development workflow for building new Kubernetes operators from initial requirements to implementation and validation.

## Agents

### operator-requirements-engineer
Conducts structured interviews to gather business requirements and produces a Product Requirements Document (PRD) for a new Kubernetes operator.

### operator-spec-designer
Consumes the PRD and generates detailed technical specifications (e.g., CR definitions and feature specs) via systematic interviews covering reconciliation logic, RBAC, and other implementation details.

### operator-implementer
Implements the operator from the PRD + technical specs, including CRD generation, controller/reconciler logic, RBAC manifests, and tests, using operator-sdk.

### operator-test-engineer
Validates the implementation against the specs by analyzing test coverage, running test suites, and producing a quality gate report with compliance verification.
