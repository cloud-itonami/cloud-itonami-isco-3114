# Business Model

## Positioning

Electronics engineering technicians run field testing, inspection, and
site-documentation workflows that are routinely tracked in spreadsheets,
paper site logs, or proprietary closed SaaS platforms that lock in the
project and expose it to vendor risk — an especially poor fit when
electronics-hazard findings and site-documentation records need to be
tamper-evident and auditable.

This open-source blueprint decouples electronics field test and inspection
coordination from vendor lock-in by providing a reference implementation of
an actor-based, governance-gated robotics system that a project operator can
deploy, modify, and own.

## Value proposition

**For a project operator:**
- Keep your own electronics test and inspection history (not rented from a
  SaaS vendor).
- Audit-logged, tamper-evident test and hazard-escalation decisions.
- Routine test data recording and inspection logging can be automated
  without loss of human oversight (escalation to a human for any
  electronics safety hazard).
- Integrated with your own identity and credential systems (no vendor API
  keys required).
- Forkable and modifiable to your specific site-documentation and
  regulatory requirements.

**For a technology partner:**
- Deploy this as a reference implementation for your customers.
- Customize the Advisor (swap `mock-advisor` for your domain-specific LLM).
- Customize the Governor rules to reflect your project's electronics-safety
  standards and regulatory constraints.
- Operate it on your own cloud or on-premises infrastructure.

## Economic model

This is **open-source software, AGPL-3.0-or-later**. There is no per-seat
licensing or per-operation fee. A project operator or technology partner
can:
1. Deploy this reference implementation (code provided).
2. Hire or retain engineers/technicians to operate it.
3. Customize the Advisor or Governor as needed for the project's
   regulatory environment.
4. Participate in the community (issues, PRs, documentation improvements).

## Regulatory & Safety considerations

**This actor is NOT an electronics actuation/energization/site-access
control system.** It does not:
- Energize, de-energize, or actuate electronics/electrical systems.
- Control power distribution.
- Grant or clear site access.

It supports **electronics test data recording, inspection logging, and
electronics-hazard escalation only** — a test/inspection robot records
readings and findings for human review; all energization/actuation/access
decisions remain under licensed technician/engineer control.

This scoping is intentional and enforced at the Governor level. It keeps
the system's attack surface small and makes it easy to certify that the
actor cannot inadvertently energize a system or suppress a hazard finding.

## Deployment architecture

### Minimal single-project deployment
```
┌─────────────────────┐
│  Project Operator     │
│   (human approval)   │
└──────────┬──────────┘
           │ (approval)
        ┌──▼──────────────────┐
        │    ElexActor          │
        │  (langgraph graph)  │
        └──┬───────────────────┘
           │
    ┌──────┴────────┬──────────────┐
    │               │              │
┌───▼────┐  ┌──────▼─────┐  ┌─────▼──┐
│ Advisor │  │  Governor  │  │ Store  │
│ (mock)  │  │  (policy)  │  │ (file) │
└─────────┘  └────────────┘  └────────┘
    │               │              │
    └───────────────┴──────────────┘
           (in-process)
```

### Scaled multi-project deployment
```
┌─────────────────────────────────────────────┐
│   Operator Console (web, shared frontend)   │
│   (authentication, human approval UI)       │
└──────────┬──────────────────────────────────┘
           │
    ┌──────┴────────┬────────────┐
    │               │            │
┌───▼────────┐  ┌──▼────────┐ ┌─┴────────┐
│ Project A  │  │ Project B │ │Project C │
│  actor pod │  │ actor pod │ │actor pod │
└──────┬─────┘  └─────┬─────┘ └──┬──────┘
       │              │          │
   ┌───┴──────────────┴──────────┘
   │
┌──▼──────────────────────┐
│  Shared audit ledger    │
│  (durable backing)      │
└─────────────────────────┘
```

### Integration points
- **Test/inspection ingestion**: field test instruments, multimeters,
  automated test equipment (ATE) → project's infrastructure.
- **Advisor**: Can be swapped out for an LLM (langchain) or domain-specific
  model.
- **Identity**: Technician/engineer approvals linked to authenticated user
  identities (OAuth, OIDC, LDAP).
- **Audit ledger**: Can be durable (Postgres, DynamoDB) or ephemeral (for
  testing).
- **Downstream actions**: Approved site-visit scheduling → work-order
  system, or email, or Slack notifications.

## Maturity & Roadmap

**Phase 1 (current):** Reference implementation with `mock-advisor`,
in-memory store, and manual testing.

**Phase 2 (future):** LLM advisor integration (langchain/claude), durable
audit ledger (Postgres), web-based operator console for approval UI.

**Phase 3 (future, if adopted):** Multi-project deployment, ATE integration
templates, work-order system integrations, regulatory compliance packs
(IEC, UL, etc.).
