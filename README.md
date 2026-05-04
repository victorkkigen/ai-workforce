# AI Employees: Treating Human and AI Workers as One Data Model

A design pattern for production systems that need humans and AI agents to coexist as first-class workers inside the same org chart, with shared concepts of role, authority, documentation, and audit.

---

## The problem

Most ERP and back-office systems were built before LLM-driven agents were practical. When a team starts adding AI workers — a support agent, an executive assistant, an operations bot — the agents end up modelled as one of three things:

1. **A user account with a fake name** ("MZURI Assistant"). Audit trails record string identities. The system has no concept of who supervises the bot, what it's allowed to approve, or which procedures govern it.
2. **A separate "bots" module** parallel to the employees system. Two org charts, two permission systems, two sets of documentation. They drift.
3. **An orchestration layer** that sits outside the business domain entirely. Convenient for engineers, opaque for the business.

None of these match how the work actually flows. An AI support worker reports to someone. It has a job description. It follows SOPs. It has approval limits. It can be promoted, retrained, or fired. These are employment concepts, not bot concepts.

## The pattern

Model humans and AI workers as instances of the same `Employee` entity, distinguished by an `employee_type` enum (`'human' | 'ai'`). AI-specific configuration lives in a sidecar table, not in the main employee record. Roles are defined separately as `Functions` — a Function is the job description; an Employee occupies one or more Functions.

```
Employee (id, type, status, primary_function, reports_to, ...)
  └── (if type='ai') AIEmployeeProfile (model_provider, approval_band_override, tool_overrides, ...)

Function (id, code, name, default_approval_band, reports_to_function, ...)
  ├── FunctionDocument (SOPs, policies, procedures attached at the role level)
  └── FunctionTool (which tools/capabilities this role is permitted to use)

EmployeeFunctionAssignment (employee, function, is_primary, dates)
```

The whole org chart — humans and AI — lives in one tree under `reports_to`. Approval authority is defined at the Function level (defaulting from the role) with optional override at the Employee level. SOPs are attached to Functions, so when a person is promoted into a role they inherit the documentation; when an AI worker is assigned to a role it's prompted with the same documents.

## Why this shape

A few decisions that fall out of the structure and turn out to matter:

**Functions, not positions.** Tenant-side ERPs usually have a `positions` table tied to org-chart slots. Functions are different: they describe *what kind of work this is*, not *which seat in the chart*. A "Customer Service Agent" Function can be occupied by three humans and one AI worker simultaneously. Positions don't compose that way.

**Sidecar instead of polymorphic columns.** The naive version puts `is_ai`, `model_provider`, `approval_band_override`, `tool_config_json` directly on the employee table. It works until you have ten AI-specific columns NULL on every human row and the table becomes unreadable. A sidecar table keeps the core employee model clean and lets the AI side evolve independently.

**Approval bands as integer ladders, not separate tables.** Some teams reach for a full `ApprovalBand` table with thresholds and rules. For a system at this stage, an integer column on the Function (with a nullable override on the AI profile) covers the cases you actually have without inventing structure you don't need. You can promote it later if reality requires.

**Audit attribution to employee records, not strings.** Every action — a ticket reply, a claim approval, a sent message — gets attributed to an `employee_id`. Humans and AI workers leave the same shape of footprint. Reporting, accountability, and retrofits to existing modules all become easier because there's one identity column instead of two.

## Retrofitting an existing system

The hardest part isn't the new tables. It's that you already have running AI features — a support agent, a conversation auto-reply, a claims handler — and they currently identify themselves with hardcoded strings. Each one needs to be migrated to attribute its actions to a real employee record without breaking in flight.

The strategy that worked:

1. **Build the new model on the platform-admin side first.** Don't touch tenant-side code or any runtime agent. Just stand up the tables, the seed data, and the admin UI to view them.
2. **Pick one runtime agent as the first retrofit.** Wire it to resolve its identity from the employee record (`employee_id`, `function_id`, approval band, allowed tools, attached SOPs) instead of hardcoded constants. Verify by running it against the dev DB.
3. **Use that one agent as the template for the rest.** Don't migrate everything at once. Each subsequent retrofit follows the same pattern; some will reveal edge cases the first one didn't.
4. **Leave the audit-ledger retrofit for last.** Existing string-based attribution can keep working until you're ready to backfill.

## What this is not

The pattern is appealing enough that it tempts overreach. A few things that look natural but are out of scope:

- **It's not a replacement for RBAC.** Functions describe role-level documentation and approval, not row-level permissions. Existing role/permission systems keep doing their job.
- **It's not a competency framework.** Skills, training events, gap analysis, and performance review are real things to build, but they sit on top of this layer. Don't conflate them.
- **It's not a tenant-facing feature on day one.** Build it for the platform admin first. Tenant rollout adds tenancy concerns (per-tenant org charts, cross-tenant AI workers) that complicate the core model. Get the model right before exporting it.
- **It's not a workflow engine.** Functions don't have execution semantics. They're declarative.

## Why it matters

When a domain expert at a customer site says "I want the AI assistant to escalate to Mary if the claim is over £5,000," there's a place in the data model for that sentence. The AI assistant is an Employee. Mary is an Employee. The £5,000 threshold is on the AI's Function, possibly overridden on its profile. Mary's authority comes from her Function. The system understands the question in the same vocabulary the customer used to ask it.

That's the thing this pattern is for. Not making AI agents work — they already work. Making them legible inside a business the way humans are.

---

*Notes from architectural design work, 2026.*
