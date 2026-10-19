# Solution Philosophy

> How I approach complex technical problems.

---

## 1. Start with the Constraint, Not the Tool

The tool is never the answer. The answer is the set of **real-world constraints** that the solution must survive within:

- Power budget (battery life, heat dissipation)
- Cost ceiling (BOM cost, licensing, maintenance)
- Environmental reality (vibration, temperature, dust, network instability)
- Operational reality (who will maintain it? what happens at 3am?)

A good solution often looks **boring** — because it's optimized for the constraint, not for elegance.

---

## 2. Full-Stack Thinking

Most hard problems in IoT / Vision systems are **not** algorithm problems. They are **integration problems**:

```
The real difficulty is rarely in one layer.
It's in the seams between layers.

    [ Camera ]          [ Algorithm ]
         │                    │
         └────────┬───────────┘
                  │
            ┌─────▼─────┐
            │ The join  │  ← failures cluster here
            │  (timing,  │
            │ power, sync)│
            └─────┬─────┘
                  │
         ┌────────┼────────┐
    [Gateway]  [Cloud]  [Client]
```

I design for how the layers meet — not just the individual components.

---

## 3. Deployability = Part of the Design

A solution that can't be deployed reliably doesn't exist.

I consider deployment from day one:
- OTA update strategy
- Field diagnostics and logging
- Graceful degradation when things fail
- What "maintainable" actually means for the operator

---

## 4. Document the "Why", Not Just the "What"

Every significant design decision should be **traceable**.

Not because it looks good in a portfolio — but because in 6 months, someone (possibly me) will need to understand why a decision was made, and whether it still holds.

Case studies in this portfolio follow this principle: **Problem → Constraints → Solution → Key Decisions**.
