# G3 Gear Desk — Warranty & Returns Triage Chatbot (MVP)

A conversational triage assistant that takes a customer's description of a gear problem and routes it to one of four outcomes: **warranty claim**, **replacement part purchase**, **self-service repair**, or **human escalation**.

Built as a vertical slice against a single brand's product line — G3 (Genuine Guide Gear) ZED 12 touring bindings — with deliberate, documented integration points for image analysis and retrieval-augmented search.

---

> ### ⚠️ This is an academic project
>
> This repository is a personal learning and portfolio exercise. It is **not** a G3 product, is not affiliated with or endorsed as a commercial offering by Genuine Guide Gear, and is not connected to any live G3 system, order database, or warranty process.
>
> Product names, part numbers and policy language are used for illustration only and may be inaccurate or out of date. **Nothing this bot outputs constitutes a real warranty determination.** Anyone with an actual G3 warranty question should contact G3 directly.
>
> G3 have kindly given permission for their product line to be used as the subject matter for this exercise. That permission covers the academic use of their catalogue as a worked example; it is not a commercial endorsement.

---

## Why this problem

Warranty and returns triage is one of the highest-volume, lowest-margin interactions in outdoor retail. Every ticket has three characteristics that make it a strong automation candidate:

1. **It's repetitive.** The same twenty or so failure modes account for the large majority of contacts.
2. **It's decidable from a small set of facts.** Product, model year, failure mode, and evidence of the failure are usually sufficient to route correctly.
3. **It's expensive to get wrong in both directions.** Approving normal wear as a warranty defect erodes margin. Rejecting a genuine defect costs a customer.

The commercial case is not "replace the CS team." It is **triage** — arrive at the human agent with the product identified, the failure mode classified, the relevant policy already retrieved, and the obvious self-service cases already deflected.

## Scope of this MVP

**In scope**

- Multi-turn conversation that gathers product, model year, and failure description
- A documented decision policy classifying the issue as *manufacturing defect*, *normal wear*, or *user-serviceable*
- Routing to one of four outcomes with a plain-language explanation of the reasoning
- Three integration stubs (see below) returning correctly-shaped data
- A golden-conversation test set with pass/fail results

**Explicitly out of scope**

- Any live system integration — orders, inventory, RMA creation, CRM
- Authentication, PII handling, or session persistence
- Actual warranty adjudication authority
- Multi-brand support (see *Scaling* below)

## Architecture

```
  Customer message
        │
        ▼
  ┌─────────────────┐
  │  Triage agent   │◄──── System prompt: decision policy
  │  (Azure OpenAI) │
  └────────┬────────┘
           │
      ┌────┴────┬──────────────┐
      ▼         ▼              ▼
 [STUB]      [STUB]        [STUB]
 Image      Knowledge      Order
 Analysis   Retrieval      Lookup
      │         │              │
      └─────────┴──────────────┘
                │
                ▼
     Routed outcome + rationale
```

### The three stubs

Each stub is a live node in the flow. It is called, its response shape is real, and the agent reasons over the result — only the data behind it is canned. This is deliberate: the point of the exercise is to show that the interface contract was designed before the features were built.

| Stub | Returns | Would be filled by |
|---|---|---|
| **Image Analysis** | `{ product, model_year, confidence, source }` | [enduro-bike-classifier-azure-ai](https://github.com/seanbrigden/enduro-bike-classifier-azure-ai) — same pattern, retrained on G3 bindings |
| **Knowledge Retrieval** | `{ passages[], part_skus[], policy_ref }` | The companion G3 ZED 12 replacement-parts RAG repo |
| **Order Lookup** | `{ order_id, purchase_date, warranty_status }` | The retailer's commerce platform |

Replacing a stub means swapping the node, not rewriting the agent. That is the whole argument.

## Tool selection

The build platform was chosen against three constraints: a working demo inside a few hours, an architecture legible to a non-technical reviewer, and honest reuse of existing work.

### ComfyUI — evaluated and rejected

ComfyUI was the original intent, and the reasoning was sound. I use it regularly for image generation and LoRA training, its node canvas is unusually good at communicating structure to a non-technical audience, and — contrary to a common assumption — it **does** run LLMs via community nodes.

It was dropped after evaluation, for reasons worth recording:

- **It's a diffusion pipeline tool.** Its execution model is a batch graph: inputs in, artifact out. Conversational state across turns is something you build *against* the tool rather than with it.
- **No shareable demo.** ComfyUI has no hosted chat endpoint. A reviewer cannot click a link; they must clone, install, and run it locally. For a portfolio artifact that is a fatal cost.
- **LLM support is community-maintained.** Custom node packs are capable but carry dependency and version risk that a few-hour build cannot absorb.
- **The canvas advantage isn't exclusive.** LLM-native node editors deliver the same visual legibility with conversation handling built in.

I'm documenting this as a rejected option rather than quietly omitting it, because the tool selection *is* part of the work. Choosing the tool you already know over the tool that fits the problem is the more common failure, and it's the more expensive one.

### What was chosen instead

An LLM-native node canvas orchestrating an Azure OpenAI model. This keeps the visual architecture story, keeps continuity with the Azure AI-900 work in the companion repos, and produces an exportable flow definition that lives in this repository as a reviewable artifact.

## Scaling to a multi-brand retailer

This MVP is a single-brand vertical slice on purpose — one catalogue, one warranty policy, unambiguous ground truth. A multi-brand retailer changes the problem in specific, identifiable ways:

- **Policy fragmentation.** Warranty terms vary by brand, by model year, and by whether the item was full-price or closeout. This moves the policy out of the system prompt and into the retrieval layer, where it belongs — the agent stops holding rules and starts citing sources.
- **Product identification becomes the bottleneck.** With one product line, the customer can name their gear. Across tens of thousands of SKUs they frequently cannot, which is precisely where the image stub stops being a demonstration and starts being load-bearing.
- **Routing gets a fourth dimension: who pays.** Brand-warranty versus retailer-goodwill versus vendor-chargeback is a commercial decision, not a technical one, and it needs to be encoded explicitly rather than left to the model.
- **Evaluation has to scale with the catalogue.** A golden set of ten conversations validates a prototype. A production system needs continuous evaluation against sampled real tickets.

None of this is speculative architecture — each item maps to a component already present as a stub here.

## Evaluation

The bot is tested against a fixed set of golden conversations covering each routing outcome plus the known-hard edge cases: ambiguous wear-versus-defect descriptions, out-of-warranty items with a purchasable part, and cases that should escalate rather than resolve.

Results and methodology are in [`/evaluation`](./evaluation).

## Limitations

- The decision policy is a reasonable reconstruction, not G3's actual warranty policy.
- Stub responses are fixed; the flow does not yet fail gracefully on stub errors.
- No adversarial testing, no prompt-injection hardening, no cost or latency benchmarking.
- Golden-set size is appropriate to a prototype and nowhere near sufficient for production confidence.

## Related portfolio work

| Repo | What it demonstrates |
|---|---|
| `enduro-bike-classifier-azure-ai` | Image classification — Azure Custom Vision |
| G3 ZED 12 parts RAG | Retrieval over a product catalogue — Azure |
| **This repo** | Conversational orchestration and integration design |

The three are designed to compose: identify the product, retrieve what applies to it, and talk to the customer about it.

---

*Built by Sean Brigden. Academic project — see disclaimer above.*
