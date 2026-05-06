# How Tool Selection Actually Works in Function-Calling Agents

**Published URL:** https://medium.com/@chalielijalem/how-tool-selection-actually-works-in-function-calling-agents-1961c1d3d222

**Question answered (Gashaw Bekele):**
> When an LLM agent has multiple tools available, what is the mechanism by which it selects one — does the model reason over tool descriptions, does the framework constrain the output format, or both? Specifically, when two tools serve overlapping purposes (like a semantic search tool and a structured query tool), what determines which one the model picks, and what makes a tool description reliably steer that choice?

---

## The Short Answer

Both — but they operate on completely different things. The model reasons over descriptions to choose **which tool** to call. The framework constrains the **output format** to be a valid structured call. These are orthogonal. You can call the wrong tool with perfectly valid JSON. That failure is invisible and produces no error.

---

## What the Model Actually Sees

Tools are not a separate routing system. When you call an LLM API with tools, the tools are serialized as structured text and injected into the model's context window alongside the conversation history. The model reads them as text — tool name, description, parameter schema — before the forward pass runs.

Tool selection is next-token prediction over this text. The same transformer weights that predict normal output tokens also predict which tool to invoke. There is no separate tool-router module.

---

## The Two Layers

**Layer 1 — Model reasoning (which tool):** The model reads the tool definitions, matches the current task to what each tool's description says it does, and outputs a tool call. This is semantic matching through language modeling. ReAct (Yao et al., ICLR 2023) made this visible: when you force the model to output a reasoning trace before acting, you see it explicitly reasoning *"I need X, tool Y handles X"* before committing. Toolformer (Schick et al., NeurIPS 2023) showed this behavior emerges from fine-tuning on examples where tool calls appear in context and produce useful results — not from explicit selection supervision.

**Layer 2 — Framework constraints (valid format):** After the model decides which tool to call, the framework ensures the output is a schema-conforming call: correct field names, required parameters, right data types. This happens through training (the model learned to output valid JSON), schema validation at the API level, and in some inference systems, constrained decoding via logit masking.

The critical distinction: Layer 2 constrains the format. It does not influence which tool is chosen. Only Layer 1 does.

---

## The Overlapping Tools Problem

ToolScope (Liu et al., 2025) measured the cost of description overlap directly: **8.8–38.6% degradation in selection accuracy** across benchmarks when tools have overlapping descriptions, compared to clearly distinct ones.

When two tools can plausibly match the same query, the model uses these signals to break the tie, in rough order of influence:

1. **Anti-trigger phrases** — "NOT when you have an exact ID" — the most powerful signal, produces active suppression
2. **Trigger phrases** — "use when searching by similarity"
3. **Tool name clarity** — `lookup_customer_by_id` vs `search_leads` is already a strong prior
4. **Parameter name overlap** — if the user says "customer ID C-1042" and the tool has a `customer_id` parameter, the model pattern-matches this
5. **Position in the tools list** — earlier tools have a slight primacy advantage

Without clear disambiguation in descriptions, the model defaults to the broader or first-listed tool — consistently and silently.

---

## What Makes a Description Reliable

The three-component pattern:

```
lookup_customer: Retrieve a single customer record by exact field value.
Use when you have a specific customer_id, email, or other exact identifier —
e.g., "customer ID C-1042", "email ana@acme.com".
NOT for fuzzy or similarity-based queries.
Prefer this over search_leads when the user provides a specific identifier.
```

**Component 1 — Trigger:** When to use this tool ("use when you have an exact field value")  
**Component 2 — Examples:** Anchors the semantic match ("customer ID, email, SKU")  
**Component 3 — Anti-trigger:** When NOT to use it ("NOT for fuzzy queries")

Anti-triggers are consistently more powerful than trigger phrases. A model that has seen "do NOT call this when X" will avoid the tool in X-like situations more reliably than it will select the tool from a trigger phrase alone. If your descriptions cannot be disambiguated — because the user hasn't provided enough signal yet — the right response is not better descriptions. It is a clarifying step: Disambiguation-Centric fine-tuning (Hathidara et al., ACL 2026) showed 27 pp improvement over GPT-4o by training models to ask for the missing signal before calling either tool.

---

## Practical Test

Before deploying a multi-tool agent, run every overlapping tool pair against 6–10 test messages with expected tool calls. 100% pass rate means descriptions are unambiguous. Any miss is a description problem. Fix the description before considering fine-tuning or retrieval — overlapping descriptions are a correctness bug, not a performance issue.

---

## Sources

1. Yao et al., *ReAct: Synergizing Reasoning and Acting in Language Models*, ICLR 2023 — https://arxiv.org/abs/2210.03629 — shows explicit reasoning traces preceding tool selection; establishes that selection is model reasoning over descriptions.

2. Schick et al., *Toolformer: Language Models Can Teach Themselves to Use Tools*, NeurIPS 2023 — https://arxiv.org/abs/2302.04761 — shows how fine-tuning creates the selection behavior from few demonstrations.

3. Liu et al., *ToolScope*, 2025 — https://arxiv.org/abs/2510.20036 — measures 8–38% accuracy degradation from overlapping descriptions; the source for the quantitative claim.