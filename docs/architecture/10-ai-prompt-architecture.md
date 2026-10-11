# 10. AI Prompt Architecture

## Provider abstraction

Brief §46 specifies OpenAI explicitly as the deployment choice, so that's the first
implementation — but it sits behind the same kind of swappable interface as `DataProvider`,
for the same reason (brief §11's spirit applies to the LLM too: don't hard-wire a vendor into
business logic):

```ts
// src/lib/ai/provider.ts
export interface LlmProvider {
  name: string;
  complete(args: {
    systemPrompt: string;
    userPrompt: string;
    model: string;
    temperature: number;
    maxOutputTokens: number;
  }): Promise<{ text: string; inputTokens: number; outputTokens: number }>;
}
```

`OpenAiProvider` is the only implementation in MVP. All prompt code calls `LlmProvider`, never
the OpenAI SDK directly.

## Model selection (cost control, brief §33)

| Prompt | Model tier | Why |
|---|---|---|
| Website Analyst (classification-heavy) | small/cheap (e.g., GPT-4o-mini class) | High volume (many pages per audit), task is close-ended classification, doesn't need frontier reasoning |
| Competitive Analyst | small/cheap | Structured comparison over already-normalized evidence |
| Revenue Analyst | small/cheap | Narrow task (flag applicable assumption keys), no arithmetic asked of the model at all (see `09-revenue-estimation.md`) |
| Executive Analyst (narrative generation) | mid-tier (e.g., GPT-4o class) | Only prompt producing client-facing prose; quality here is what the GM actually reads, worth the marginal cost |

Model names are config, not hardcoded per-call, so a pricing/quality change is a one-line
change in `src/lib/ai/models.ts`, versioned alongside `prompt_version`.

## The four prompts (brief §15)

Every prompt's **system message** includes, verbatim, the brief's required guardrail (§14):

> "Use only the supplied evidence. If evidence is insufficient, explicitly state that evidence
> is insufficient. Do not invent facts, competitors, traffic, revenue, inventory, prices,
> conversion rates, or statistics not derivable from the supplied evidence. Do not claim legal
> compliance or guaranteed outcomes."

### Prompt 1 — Website Analyst
- **Input:** URL, extracted page text/structure (nav, headings, detected CTAs, meta tags),
  deterministic technical findings (page speed metric, broken-link list, mobile viewport
  check) — all already computed by code, not asked of the model.
- **Output schema:**
  ```ts
  z.object({
    findings: z.array(z.object({
      category: FindingCategorySchema,
      observation: z.string(),
      evidenceRefs: z.array(z.string()), // must cite supplied evidence ids, nothing else
      severity: SeveritySchema,
    })),
    missingInformation: z.array(z.string()),
    confidence: z.number().min(0).max(100),
  })
  ```
- The model classifies/explains; `evidenceRefs` must resolve to ids actually in the supplied
  evidence set — any ref that doesn't resolve fails validation (see retry policy below) rather
  than being silently accepted, which is the concrete mechanism preventing invented evidence.

### Prompt 2 — Competitive Analyst
- **Input:** client evidence set + each competitor's evidence set (all already collected,
  normalized, deterministic).
- **Output schema:**
  ```ts
  z.object({
    competitiveGaps: z.array(GapSchema),
    advantages: z.array(GapSchema),
    opportunities: z.array(z.object({ title: z.string(), evidenceRefs: z.array(z.string()) })),
  })
  ```
- Explicitly forbidden (enforced by system prompt + spot-checked in AI contract tests, brief
  §34): inventing a competitor not present in the supplied `competitors` evidence set.

### Prompt 3 — Revenue Analyst
- **Input:** deterministic findings + calculated metrics + the Assumption Library's current
  key names (not values — the model never sees or chooses numeric benchmark values, only which
  *category* of assumption applies).
- **Output schema:** no numeric revenue fields (see `09-revenue-estimation.md`):
  ```ts
  z.object({
    applicableAssumptionKeys: z.array(z.string()),
    additionalDataNeeded: z.array(z.string()),
    confidence: z.number().min(0).max(100),
  })
  ```

### Prompt 4 — Executive Analyst
- **Input:** finalized findings, scores, revenue ranges (already computed), evidence summaries.
- **Output:** executive summary, top-3 opportunities (by `opportunityScore`, pre-sorted by
  code, not chosen by the model), recommended actions, concise explanations.
  ```ts
  z.object({
    executiveSummary: z.string(),
    topOpportunities: z.array(z.object({ findingId: z.string(), narrative: z.string() })).max(3),
    recommendedActions: z.array(z.string()),
  })
  ```
- `findingId` must reference a real finding id passed in; narrative text is purely
  `AiInterpretation` category and never feeds back into any score or dollar figure.

## Structured output enforcement

```ts
// src/lib/ai/structured-call.ts (sketch)
export async function structuredCall<T>(args: {
  provider: LlmProvider;
  promptVersion: string;
  systemPrompt: string;
  userPrompt: string;
  schema: z.ZodType<T>;
  model: string;
}): Promise<T> {
  const attempt = async (correctionNote?: string) => {
    const { text, inputTokens, outputTokens } = await args.provider.complete({
      systemPrompt: correctionNote ? `${args.systemPrompt}\n\n${correctionNote}` : args.systemPrompt,
      userPrompt: args.userPrompt,
      model: args.model,
      temperature: 0,
      maxOutputTokens: 2000,
    });
    await recordAiUsage({ model: args.model, inputTokens, outputTokens, promptVersion: args.promptVersion });
    return args.schema.parse(JSON.parse(text)); // throws ZodError on invalid shape/values
  };

  try {
    return await attempt();
  } catch (err) {
    // one retry with the validation error fed back as a correction instruction
    return await attempt(`Your previous response was invalid: ${describeZodError(err)}. Return corrected JSON only.`);
    // if this also throws, the caller (job step) catches it and the stage degrades per 06-audit-pipeline.md
  }
}
```

- Temperature `0` everywhere — these are classification/extraction tasks, not creative writing,
  and determinism matters for reproducibility (brief §23).
- Exactly one retry-with-correction (brief §16) — not an unbounded loop, to bound cost and
  latency; a second failure is treated as that stage's final failure and degrades per the
  pipeline's required/degradable table.
- `recordAiUsage` writes to `ai_usage` on every attempt (success or failure), so a model that
  burns tokens failing validation twice is still cost-tracked — brief §33's "cost per audit"
  tracking must include failed/retried calls, not just successful ones.

## Prompt versioning

Each prompt lives in its own file under `src/prompts/<name>/v<N>.ts` exporting
`{ version: "v1", systemPrompt, userPromptTemplate }`. `prompt_version` stamped on the audit
(see `06-audit-pipeline.md`) is a composite string like `website-analyst@v1,competitive@v1,
revenue@v1,executive@v1` — bumping any one prompt's wording creates a new `vN` file (the old
one is never edited in place) and a new composite version string, so re-running the exact logic
behind a historical audit is always possible by checking out that prompt version.

## AI contract tests (brief §34)

Fixed evidence fixtures under `src/tests/ai-contract/fixtures/` are fed through each prompt
against the real provider (recorded/replayed in CI to control cost — see
`13-testing-architecture.md`), asserting: valid JSON per schema, no `evidenceRefs` pointing
outside the fixture's evidence set, required fields present, score fields within `[0,100]`, and
— critically — a fixture engineered to have *insufficient* evidence produces the explicit
"insufficient evidence" output rather than a confident-sounding guess.
