# base-claude-md

A base `CLAUDE.md` for a multi-platform telehealth and clinic compliance system,
developed and refined throughout 2026. This is the early-August version, tuned
for Opus 5. All business-specific content — company and product names, domains,
credentials, and domain logic — has been redacted; what remains is the
structure and the operating rules.

## Approach

The focus is a **minimalist** `CLAUDE.md` that leans on model intelligence
rather than enumerating procedure. Concretely:

- **Tools and skills load on demand.** The document points at skills by name
  (`staging`, `shipping`, `testing`, `migrations`, …) and lets the model decide
  when to pull one in, instead of inlining every workflow up front. Detail lives
  in the skill files; this file stays a router.
- **Verification depth is a judgment call.** A single blast-radius rule near the
  top ("light path" vs. "full protocol") governs the rest of the file, so the
  model scales rigor to what a change can actually break rather than applying
  one checklist to everything.
- **Implementation and delegation depth are likewise delegated.** Guidance on
  when to spawn agents, how tightly to scope them, and when to just do the work
  inline is stated as principles — scope by mechanism, not by list — rather than
  as a decision table.
- **Hard-won specifics stay explicit.** The narrow rules that survived are the
  ones a model won't infer: prod-mutation gates, footguns in specific libraries,
  and bugs that pass every local test.

Standing constraints (prod safety, never push to `main` as a subagent) are
stated once and scoped to govern every section, so they don't need repeating.
