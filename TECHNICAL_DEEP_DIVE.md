# Technical Notes: Prompt Optimization Tradeoffs

> These are working notes documenting implementation decisions, observations, and open questions. This is not a formal paper.

**Note on Cost Estimates**: All token pricing mentioned throughout this document are rough approximations based on publicly available information at time of writing. Actual costs vary significantly. See disclaimer at end of document.

---

## 1. Problem Framing

### The Question

When users write prompts for LLMs, they typically write them naively—short, underspecified, missing context. The prompt engineering community has developed various "best practices" (role assignment, chain-of-thought, examples), but these come with token costs.

**Core tension**: More structured prompts may produce better outputs, but they also cost more. Is there a principled way to navigate this tradeoff?

### Why This Is Hard

1. **"Quality" is undefined**: What makes an LLM output good? Accuracy? Helpfulness? Style? It depends entirely on context.

2. **No ground truth**: Unlike supervised learning, we don't have labeled datasets of "good prompts" and "bad prompts."

3. **Model-dependent**: What works for GPT-4 may not work for Claude or Llama.

4. **User-dependent**: Expert users may need less scaffolding than novices.

5. **Task-dependent**: Simple queries need different treatment than complex reasoning tasks.

---

## 2. Design Decisions

### Why Rule-Based?

I chose rule-based heuristics over learned approaches because:

1. **Interpretability**: I can understand exactly why the system makes each decision.
2. **No training data needed**: I don't have a labeled dataset of prompts.
3. **Speed**: Regex matching is fast; no inference required.
4. **Debugging**: When it fails, I can trace the failure to a specific rule.

**Tradeoff acknowledged**: Rule-based systems are brittle. They fail on edge cases and can't generalize beyond their explicit rules.

### Why Client-Side?

1. **Privacy**: No prompts sent to external servers.
2. **Latency**: No network round-trip.
3. **Simplicity**: No backend to maintain.

**Tradeoff**: Can't use actual LLM calls to evaluate or improve prompts.

### Why Token Budget?

Early versions of this project added too many tokens indiscriminately. A user's simple 10-token prompt would become a 500-token behemoth. This felt wrong—I was "optimizing" in a way that ignored cost.

The token budget forces explicit tradeoff consideration: "Given N tokens to spend, what's the highest-value transformation?"

---

## 3. Implementation Details

### Intent Detection

```javascript
const INTENT_PATTERNS = {
  coding: ['code', 'function', 'debug', 'api', 'error', 'bug', 'script'],
  creative: ['story', 'write', 'poem', 'imagine', 'fiction'],
  analysis: ['analyze', 'compare', 'evaluate', 'data', 'research'],
  explain: ['explain', 'what is', 'how does', 'why', 'define'],
  task: ['create', 'make', 'build', 'generate', 'list', 'plan']
};
```

**How it works**: Scan input for keywords, return first matching category.

**Failure modes**:
- "Write code for a story generator" → matches "code" first, misses creative intent
- "Analyze why my code has bugs" → ambiguous between analysis and coding
- Non-English prompts → no matches, defaults to generic "task"

**Measured accuracy**: Informal testing on ~50 prompts suggested ~75% accuracy. This is not a rigorous evaluation.

### Gap Detection

Ten boolean checks:

| Gap | Check | Limitation |
|-----|-------|------------|
| No role | Regex for "you are", "act as" | Misses implicit roles |
| Vague | Length < 40 chars | Arbitrary threshold |
| No format | Regex for "json", "table", etc. | Misses format needs that aren't explicit |
| No context | Length < 80 and no "because/given" | Very crude |
| No constraints | No "don't/avoid/must/only" | Misses implicit constraints |

**Observation**: These checks have high recall (they catch most issues) but lower precision (they flag issues that may not matter).

### Token Counting

```javascript
function countTokens(text) {
  return Math.ceil(text.length / 4);
}
```

**Why this approximation?**
- Actual tokenizers are model-specific
- Loading a tokenizer adds complexity
- For relative comparisons, approximation is sufficient

**Error analysis**: Compared against OpenAI's tiktoken on 20 sample prompts:
- Average error: 12%
- Max error: 28% (on a code-heavy prompt with many symbols)
- Consistently underestimates tokens for code

### Quality Scoring

```javascript
function calculateQuality(original, optimized, gaps) {
  let clarity = 40 + (hasRole ? 15 : 0) + (hasStructure ? 10 : 0);
  let specificity = 30 + (lengthRatio * 25);
  let structure = 30 + (hasHeaders ? 30 : 0);
  let actionability = 40 + (hasConstraints ? 20 : 0);
  
  return (clarity + specificity + structure + actionability) / 4;
}
```

**This is entirely heuristic.** There is no validation that these weights are meaningful or that higher scores correlate with better LLM outputs.

I chose to include this metric because users expect feedback, but I'm uncomfortable with how much false precision it implies.

---

## 4. Observations During Development

### Things That Surprised Me

1. **Minimal changes often suffice**: Adding just a role tag ("You are a Python expert.") to a coding prompt felt surprisingly effective in informal testing. The elaborate scaffolding in "maximum" mode may be overkill for many tasks.

2. **Token costs add up fast**: Chain-of-thought scaffolding alone adds 80-150 tokens. For GPT-4 at $0.03/1K tokens, that's real money at scale.

3. **Intent detection is the weak link**: Most failures trace back to misclassified intent, which causes wrong techniques to be applied.

4. **Users write very short prompts**: In my informal sample, median prompt length was 12 tokens. There's clearly room to add value—but how much?

### Things I'm Still Uncertain About

1. **Does structural improvement help?** I've assumed that adding role/format/constraints improves outputs, but I haven't measured this.

2. **Is the cost-quality tradeoff smooth?** Maybe outputs are binary (works or doesn't work) rather than gradually improving with more tokens.

3. **Do users want this?** Maybe expert users prefer to write their own prompts. Maybe novices need guidance, not automation.

4. **Model differences**: I've mostly tested mentally against GPT-4. Claude, Gemini, and open models may respond differently to the same prompts.

---

## 5. What Would Rigorous Evaluation Look Like?

If I were to properly evaluate this system, I would:

### Dataset
- Collect 500+ prompts across 7 intent categories
- Include ground-truth intent labels
- Include human judgments of prompt quality

### Evaluation Protocol
1. Run each prompt (original and optimized) through multiple LLMs
2. Have human evaluators rate outputs on:
   - Correctness/accuracy
   - Helpfulness
   - Appropriate length
   - Following instructions
3. Compare ratings: original vs. optimized
4. Segment by intent category and optimization mode

### Metrics
- Win rate: % of cases where optimized prompt produces better output
- Cost efficiency: quality improvement per additional token
- Failure rate: % of cases where optimization hurts

### Controls
- Test against null hypothesis: random token addition
- Ablation: test each technique in isolation
- Cross-model: test on GPT-4, Claude, Gemini, Llama

**I have not done this.** The current project is a prototype for exploring the space, not a validated system.

---

## 6. Related Work (Partial)

These papers informed my thinking, though I don't claim to implement them fully:

- **Wei et al. (2022)** — Chain-of-Thought: Showed that "think step by step" improves reasoning. I include CoT scaffolding but haven't measured its isolated effect.

- **Yao et al. (2023)** — Tree of Thoughts: Proposed branching exploration. My "maximum" mode gestures at this but doesn't actually implement tree search.

- **Jiang et al. (2023)** — LLMLingua: Prompt compression. Inspired my budget system, but they use learned compression while I use naive truncation.

- **Zhou et al. (2023)** — Large Language Models Are Human-Level Prompt Engineers: Showed that LLMs can optimize prompts. Raises the question: should optimization itself be done by an LLM?

---

## 7. Honest Assessment

### What This Project Does Well
- Provides a concrete framework for thinking about prompt optimization tradeoffs
- Runs fast with no dependencies
- Makes token costs visible
- Forces explicit consideration of budget

### What This Project Does Poorly
- No actual validation of quality improvements
- Brittle rule-based detection
- False precision in quality scores
- Doesn't account for conversation context
- English-only

### What I Would Do Differently
1. Start with a proper evaluation framework before building
2. Use a small LLM for intent detection instead of keywords
3. Build in A/B testing from the start
4. Focus on fewer techniques, evaluated deeply, rather than many techniques evaluated shallowly

---

## 8. Small-Scale Qualitative Evaluation (n≈25)

To sanity-check the system, I ran a limited exploratory evaluation:

**Setup**:
- 25 manually-written prompts across 5 categories
- Original and optimized (balanced mode) submitted to GPT-4
- Outputs compared qualitatively by me (not blinded)

**Rough observations**:
- ~16/25: Optimized seemed better
- ~5/25: No meaningful difference  
- ~4/25: Optimized was worse

**This proves nothing.** n=25 is too small, evaluation was subjective, single model tested. But it suggests the approach might have *some* validity worth investigating further.

---

## 9. Surprising Finding: When Optimization Hurts

~15% of cases where optimization made things worse:

1. **Simple factual queries**: "What is the capital of France?" + scaffolding → unnecessary verbosity
2. **Over-constrained creative tasks**: Maximum mode → formulaic, restricted outputs
3. **Intent misclassification**: Wrong techniques applied → confused results

**Implication**: Predicting when NOT to optimize may be harder than optimization itself.

---

## 10. Open Questions

1. Is there a universal "good prompt" structure, or is it entirely task-dependent?
2. At what token threshold do diminishing returns kick in?
3. Can we predict when NOT to optimize?
4. Would an LLM-based optimizer outperform rule-based approaches? At what cost?

---

*I built this to understand the problem, not because I have the answer.*

— Charanpreet Singh, 2025

---

## Appendix: Cost Estimation Disclaimer

All token costs referenced in this project are **rough estimates only**:

- GPT-4 ~$0.03/1K tokens (input) - *estimate*
- GPT-3.5 ~$0.002/1K tokens - *estimate*
- Claude ~$0.015/1K tokens - *estimate*
- Gemini ~$0.001/1K tokens - *estimate*

**These figures are not guaranteed to be accurate.** Actual pricing:
- Varies by provider and model version
- Changes frequently
- Differs by account tier, region, and usage volume
- Often has different rates for input vs output tokens

Always consult official pricing documentation for accurate cost calculations.
