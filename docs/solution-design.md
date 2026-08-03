# Solution Design — G3 Gear Desk

> Academic project. See root README disclaimer.

## Repository structure

```
g3-gear-desk/
├── README.md                        Disclaimer, problem, architecture, tool selection
├── .env.example                     Variable names only, no values
├── .gitignore                       Excludes .env
│
├── docs/
│   ├── SOP-build.md                 Step-by-step build runbook
│   ├── solution-design.md           This file
│   └── decision-log.md              Choices made and why
│
├── prompts/
│   └── system-prompt.md             The triage decision policy
│
├── flow/
│   └── g3-gear-desk-flow.json       Exported Flowise flow — makes this reproducible
│
├── stubs/
│   ├── image_analysis.json          Fixed response + contract
│   ├── knowledge_lookup.json
│   └── order_lookup.json
│
├── evaluation/
│   ├── golden-conversations.md      Ten test conversations
│   └── results.md                   Pass/fail, dated, including failures
│
└── assets/
    ├── canvas.png                   The node graph screenshot
    └── conversation-*.png           Example exchanges
```

Two things a reviewer will look for and most portfolio repos lack: an exported flow definition that lets someone else reproduce the build, and an evaluation folder that records failures as well as passes.

## Stub contracts

Each stub is a live tool call. The agent invokes it, receives correctly-shaped JSON, and reasons over the result. Only the data is fixed.

**`image_analysis`** — called when a photo is present.

```json
{
  "product": "G3 ZED 12",
  "model_year": 2022,
  "component_detected": "heel_piece",
  "confidence": 0.0,
  "source": "STUB — would be served by enduro-bike-classifier-azure-ai pattern, retrained on G3 bindings"
}
```

`confidence: 0.0` is deliberate. A stub returning a plausible-looking `0.94` invites a reviewer to think the model exists. Zero, plus an explicit `source`, is honest and still exercises the contract.

**`knowledge_lookup`** — called when parts, prices, or policy text are needed.

```json
{
  "query_echo": "heel riser ZED 12",
  "passages": [
    { "text": "Heel riser assemblies are a wear item...", "policy_ref": "WARRANTY-3.2" }
  ],
  "parts": [
    { "sku": "STUB-0000", "name": "ZED heel riser", "price_cad": 0.00, "in_stock": true }
  ],
  "source": "STUB — would be served by the G3 ZED 12 parts RAG repo"
}
```

**`order_lookup`** — called when warranty period must be established.

```json
{
  "order_id": "STUB-ORDER",
  "purchase_date": "2023-11-14",
  "warranty_status": "in_warranty",
  "source": "STUB — would be served by the retailer's commerce platform"
}
```

Swapping a stub for the real service means replacing the node. The agent, the prompt, and the evaluation set are unchanged. That property is the argument this repository exists to make.

## Design decisions

**The agent has no authority.** It classifies and routes; it never approves. This is a deliberate limit, not a capability gap. Any system that can approve a warranty claim needs audit logging, appeal handling, and an accountable owner — none of which belong in an MVP, and all of which are cheaper to add than to retrofit around an agent that already had authority.

**The safety rule is unconditional and sits first.** Release-related failures on a touring binding are an injury risk. There is no version of this system that should reason about whether an unexpected release was "probably fine". The rule is placed before the classification logic so it cannot be reached through a classification path.

**Wear is stated plainly, not hedged.** The instinct is to soften a rejection. Softening produces a customer who files anyway, waits three weeks, and is rejected by a human — a worse experience than a clear no in the first minute.

**Pushback escalates rather than re-argues.** A model that defends its own classification under pressure will either capitulate or entrench, and both are wrong. Disagreement is a signal that a person should be involved.

**Single brand, deliberately.** One catalogue and one policy give unambiguous ground truth, which is what makes the evaluation set meaningful. See the README for what changes at multi-brand scale.
