# SOP — Building the G3 Gear Desk MVP

**Audience:** me, and anyone reviewing whether this was structured sensibly.
**Assumed knowledge:** none beyond an Azure account and the ability to install software.
**Target duration:** 2 hours. Stage timings are budgets, not estimates — if a stage overruns, cut scope rather than the stage after it.

> This is an academic project. See the disclaimer in the root README.

---

## What you are building

A chat interface that triages a G3 ZED 12 binding problem and routes it to one of four outcomes. The model runs in Azure. The orchestration runs in Flowise, a node-canvas tool that talks to Azure over the OpenAI-compatible API.

Two systems, one connection between them. That connection is the only genuinely fiddly part, and it is Stage 2.

---

## Stage 0 — Prerequisites (10 min)

| Need | Notes |
|---|---|
| Azure subscription | Pay-as-you-go is fine. This project will cost cents, not dollars — but it is not free, and you should set a budget alert. |
| Docker Desktop **or** Node.js 18+ | Either will run Flowise. Docker is the more reliable path. |
| A GitHub repo | Create it empty; you'll push at Stage 6. |

**Set a spend cap before anything else.** Azure portal → Cost Management → Budgets → create a budget of $10 with an alert at 50%. This takes two minutes and removes the background anxiety of an unfamiliar cloud bill.

---

## Stage 1 — Provision the model in Azure (25 min)

Microsoft renamed this platform recently. You may see "Azure OpenAI", "Azure AI Foundry", and "Microsoft Foundry" used interchangeably in documentation and search results. They refer to the same thing. The current portal is **foundry.ai.azure.com**.

**1.1 — Create the resource**

Azure portal → Create a resource → search "Azure OpenAI" → Create.

- Resource group: create a new one, `rg-g3-gear-desk`. Keeping the project in its own group means you can delete the whole thing in one click when you're done.
- Region: pick one with model availability. East US or Sweden Central are safe defaults; if the model you want isn't listed at Stage 1.2, region is the usual reason.
- Pricing tier: Standard S0.
- Leave networking and identity at defaults. This is a demo, not a production deployment, and over-configuring here is where hours disappear.

Creation takes 2–3 minutes.

**1.2 — Deploy a model**

Open the resource → *Go to Microsoft Foundry portal* → Deployments → Deploy model.

- Choose a current general-purpose chat model. A mini/small tier is correct here — triage is not a reasoning-heavy task, and the cheaper model keeps your test loop fast.
- **Deployment name: `g3-triage`.** You are inventing this name. Write it down.
- Deployment type: Standard.
- Capacity: the minimum offered.

> ### The one mistake everyone makes
> Azure separates the *model* (e.g. `gpt-4o-mini`) from the *deployment name* (whatever you typed). Every downstream tool wants the **deployment name**. Supplying the model name produces a 404 that reads like an authentication problem and isn't. If Stage 2 fails, check this first.

**1.3 — Collect four values**

Resource → Keys and Endpoint. Copy into a scratch file:

| Value | Looks like | Where |
|---|---|---|
| API key | long hex string | Keys and Endpoint (either key works) |
| Instance name | `myresource` — the subdomain only, not the full URL | from the endpoint `https://myresource.openai.azure.com/` |
| Deployment name | `g3-triage` | what you typed in 1.2 |
| API version | e.g. `2024-10-21` | Foundry portal, on the deployment's sample code |

**Do not commit these to the repo.** They go in a local `.env` which `.gitignore` excludes. The repo contains `.env.example` with the keys blank — that is the artifact that shows you know the difference.

**Stage 1 exit test:** in the Foundry portal's Chat playground, select your deployment and send "hello". If it replies, Azure is done and everything after this is orchestration.

---

## Stage 2 — Stand up Flowise (15 min)

```bash
docker run -d --name flowise -p 3000:3000 flowiseai/flowise
```

Then open `http://localhost:3000`.

(Node alternative: `npx flowise start`.)

**2.1 — Add the Azure credential**

Flowise → Credentials → Add → *Azure ChatOpenAI*. It asks for exactly the four values from Stage 1.3. This is the whole integration.

**Stage 2 exit test:** build a throwaway flow — Chat Model (Azure ChatOpenAI) → Conversation Chain — and send "hello" in the chat pane. A reply means the connection is live.

If it errors: deployment name (see above), then API version, then region availability, in that order. Do not proceed past this point with a broken connection; everything downstream assumes it.

---

## Stage 3 — Build the triage flow (30 min)

Canvas layout, left to right:

```
Buffer Memory ──┐
                ├──> Conversational Agent ──> Chat output
Azure ChatOpenAI┘         │
                          ├── Tool: image_analysis   [STUB]
                          ├── Tool: knowledge_lookup [STUB]
                          └── Tool: order_lookup     [STUB]
```

**3.1** Drop **Conversational Agent** (it supports tools and image upload; a plain Conversation Chain does not).
**3.2** Attach your **Azure ChatOpenAI** node and a **Buffer Memory** node.
**3.3** Paste the contents of `prompts/system-prompt.md` into the agent's system message field.
**3.4** Add three **Custom Tool** nodes. Each returns fixed JSON — the shapes are specified in `docs/solution-design.md`. Give each a description that tells the model when to call it; the description is what drives tool selection, so it matters more than the code inside.

**Stage 3 exit test:** "my ZED 12 heel riser snapped" should produce a classification and a routed outcome, not a generic apology.

---

## Stage 4 — Evaluate (20 min)

Run the ten conversations in `evaluation/golden-conversations.md` verbatim. Record pass/fail and the actual routed outcome in the results table.

**Expect failures on the first pass.** Two or three of the edge cases will misroute. Fix them by tightening the system prompt, then re-run the full set — not just the ones that failed, because prompt changes cause regressions elsewhere. That re-run discipline is the thing worth demonstrating.

Record the final results honestly, including anything still failing. A documented 8/10 with the two failures explained is a stronger artifact than an undocumented 10/10.

---

## Stage 5 — Capture the evidence (10 min)

- Screenshot the full canvas — this is your LinkedIn image, so zoom so the node labels are legible.
- Screenshot two or three good exchanges, including one escalation.
- Flowise → the flow's settings → **Export as JSON**. Save to `flow/g3-gear-desk-flow.json`. This is what makes the repo reproducible rather than decorative.

---

## Stage 6 — Publish (10 min)

```bash
git add . && git commit -m "G3 Gear Desk MVP" && git push
```

Before pushing, confirm: `.env` is gitignored, no keys in the flow JSON export (Flowise stores credentials separately, but check), disclaimer present in the README.

---

## If you overrun

Cut in this order:

1. Order Lookup stub — the weakest of the three, drop to two
2. Golden conversations from ten to five, keeping both escalation cases
3. Buffer memory — single-turn triage still demonstrates the architecture

**Never cut:** the README, the canvas screenshot, the exported flow JSON. Those three are what a reviewer actually consumes.
