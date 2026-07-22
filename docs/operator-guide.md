# Operator Guide

This guide is for project operators and technology partners deploying the
electronics engineering technician field test and inspection actor.

## Operational workflow

### 1. Project registration

Before the actor can handle any operations, the electronics
site/installation must be registered in the store:

```clojure
(def store (store/mem-store
  {:projects {:project-001 {:project-id :project-001
                            :name "Substation Panel Retrofit"
                            :location "Site A, Building 3"}}}))
```

The Governor requires the project record to exist for any operation to
proceed.

### 2. Submit an operational request

A technician or a test-equipment integration submits a request:

```clojure
(def request {:project-id :project-001
              :op :draft-test-record
              :stake :low
              :payload {:test "continuity-check"
                        :result :pass}})
```

Allowed operations:
- `:draft-test-record` — routine test/measurement data recording.
- `:log-inspection-data` — inspection finding logging.
- `:flag-safety-hazard` — surface an electronics safety hazard; **ALWAYS
  escalates**.
- `:schedule-site-visit` — site-visit scheduling.

### 3. Run the actor

```clojure
(require '[elex.actor :as actor])

(def graph (actor/build-graph {:store store}))

(def result (actor/run-request! graph request {} "thread-001"))
;; Result: {:state {:record {...} :audit [...]}
;;          :events [...]
;;          :status :done|:interrupted
;;          :frontier ...}
```

**Possible outcomes:**

- **`:status :done`** (no escalation required)
  - The request succeeded and was committed to the store.
  - Check `:state :record` for the persisted record.

- **`:status :interrupted`** (waiting for human approval)
  - The proposal was flagged for escalation (either
    `:flag-safety-hazard` or low advisor confidence).
  - A human technician/engineer must review the proposal and decide
    whether to approve or deny.
  - Use `actor/approve!` to resume (see step 4).

- **`:status :hold`** (rejected, no escalation)
  - The Governor rejected the proposal permanently (hard violation).
  - Examples: unregistered project, non-`:propose` effect.
  - No recovery possible; the proposal is discarded.

### 4. Human approval (if interrupted)

If the actor returned `:status :interrupted`, a human must approve:

```clojure
(def approval-result (actor/approve! graph "thread-001"))
;; This resumes the interrupted request and commits it.
```

After approval, the record is committed and the actor returns
`:status :done`.

### 5. Audit & compliance

All decisions are logged in the audit ledger:

```clojure
(store/ledger store)
;; Returns: [{:node :advise :request {...} :proposal {...}}
;;           {:node :govern :verdict {...}}
;;           {:disposition :request-approval ...}
;;           {:node :commit :record {...}}]
```

Export the ledger regularly for compliance audits and incident
investigation.

## Customizing the Advisor

The default `mock-advisor` is deterministic and suitable for testing. For
production:

1. Implement the `Advisor` protocol:
   ```clojure
   (deftype LLMAdvisor [model]
     Advisor
     (-advise [_ store request]
       ;; Call your LLM to propose an action.
       ;; Always return :effect :propose.
       ;; Return :confidence 0.0 on parse failure (forces escalation).
       ))
   ```

2. Pass it to `build-graph`:
   ```clojure
   (actor/build-graph
     {:store store
      :advisor (LLMAdvisor. your-model)})
   ```

## Customizing the Governor

The Governor policy is in `src/elex/governor.cljc`. If your project has
different safety rules:

1. Modify the `:escalate?` logic (e.g., additional ops that require human
   approval).
2. Modify the `:hard?` violations (e.g., additional prerequisites that
   always reject).
3. Adjust `confidence-floor` if your advisor has different reliability
   metrics.
4. Document why the change is needed in a comment or ADR.

**Important:** Do not weaken the hard invariants:
- `:no-project` — always reject unregistered projects.
- `:no-actuation` — always require `:effect :propose`.

## Integration with external systems

### Test equipment ingestion
Your automated test equipment (ATE) or field test instrument submits
requests to the actor:
```clojure
(actor/run-request! graph
  {:project-id :project-001
   :op :log-inspection-data
   :payload {:finding "loose terminal" ...}}
  {}
  "ate-thread-001")
```

### Work-order system
When the actor commits a `:schedule-site-visit` proposal, push it to your
work-order system:
```clojure
(when (= :schedule-site-visit (:op (:record state)))
  (work-orders/create-visit (:record state)))
```

### Alerting / escalation
When `:flag-safety-hazard` is submitted, the actor escalates to human
approval:
```clojure
(if (= :interrupted (:status result))
  (send-alert-to-operator "Electronics hazard flagged, awaiting approval"))
```

## Troubleshooting

**Q: My request returned `:status :hold`. Why?**
A: The Governor rejected it as a hard violation. Check the `:verdict` in
the audit ledger for details. Common causes:
- Project not registered.
- Proposal `:effect` is not `:propose`.

**Q: My request returned `:status :interrupted`. What do I do?**
A: A human technician/engineer must review the proposal and call
`actor/approve!` to resume. This is the intended flow for escalations
(electronics hazards, low-confidence advisors).

**Q: How do I export the audit ledger for compliance?**
A: Call `(store/ledger store)` and serialize to JSON or CSV. The ledger is
append-only and tamper-evident.

**Q: Can I integrate with an LLM?**
A: Yes. Implement the `Advisor` protocol and swap `mock-advisor` for your
LLM advisor. Always return `:confidence 0.0` on parse failures (forces
escalation, never fabricated confidence).

## Further reading

- [`README.md`](../README.md) — project overview and design rationale.
- [`src/elex/governor.cljc`](../src/elex/governor.cljc) — hard/escalation
  invariants.
- [`src/elex/actor.cljc`](../src/elex/actor.cljc) — StateGraph wiring and
  flow.
