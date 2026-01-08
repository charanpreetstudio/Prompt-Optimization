# Studying Cost-Quality Tradeoffs in Prompt Engineering

> **Positioning**: This is a research exploration, not a product. I built this to understand prompt optimization behavior—its limits, failure modes, and tradeoffs. The findings are preliminary and the methodology has known gaps. I'm sharing it because the questions feel worth investigating, not because I've solved them.

---

## Research Questions

1. Can rule-based heuristics detect prompt intent with reasonable accuracy? Where do they fail?
2. What is the relationship between token overhead and prompt quality? Is it linear, logarithmic, or task-dependent?
3. When does optimization *hurt*? Can we predict failure cases?
4. Is there a principled "quality-to-cost ratio" for prompts, or is this concept fundamentally flawed?

---

## What This Project Does

A client-side tool that:
- Detects prompt intent via keyword matching (~7 categories)
- Identifies structural gaps (missing role, format, constraints)
- Applies rule-based transformations (role injection, CoT scaffolding, etc.)
- Estimates token count and cost
- Computes heuristic quality scores

**Important**: This focuses on **structural prompt properties** (length, formatting, explicit constraints), not **semantic correctness**. Whether structurally-improved prompts produce better LLM outputs remains an open question I have not rigorously validated.

---

## Methodology

### Approach
- **Intent detection**: Keyword matching against 7 categories
- **Gap analysis**: 10 rule-based checks (e.g., "does prompt contain role assignment?")
- **Transformation**: Template-based prompt augmentation
- **Token estimation**: `tokens ≈ characters / 4` (approximate, not exact)
- **Quality scoring**: Heuristic based on structural features (NOT validated against LLM output quality)

### Key Limitation
Quality scores measure *structural completeness*, not *semantic effectiveness*. A prompt scoring 85 is not proven to produce better outputs than one scoring 70. Validating this would require systematic human evaluation of LLM outputs, which I have not conducted.

---

## Small-Scale Qualitative Evaluation (n≈25)

To sanity-check the system, I ran a limited exploratory evaluation:

**Setup**:
- 25 manually-written prompts across 5 categories (coding, creative, analysis, explanation, task)
- Each prompt run through optimization (balanced mode)
- Original and optimized versions submitted to GPT-4 (temperature=0.7)
- Outputs compared qualitatively (not scored numerically)

**Observations** (not statistical claims):
- ~16/25 optimized prompts produced outputs I subjectively preferred
- ~5/25 showed no meaningful difference
- ~4/25 optimized prompts performed *worse* (see "Surprising Finding" below)

**Caveats**:
- n=25 is too small for statistical significance
- "Preference" was my subjective judgment, not blinded evaluation
- Single model (GPT-4), single temperature setting
- No inter-rater reliability

This evaluation is exploratory. It suggests the approach *might* help for some prompts, but proves nothing definitively.

---

## Surprising Finding: When Optimization Hurts

In ~15% of test cases, optimized prompts performed worse than originals. Patterns I noticed:

1. **Simple factual queries**: Adding role/CoT scaffolding to "What is the capital of France?" produced unnecessarily verbose outputs. The original prompt was already optimal.

2. **Creative tasks with over-constraint**: Maximum-mode optimization added so many constraints that creative outputs felt formulaic and restricted.

3. **Ambiguous intent**: When my intent detector misclassified prompts (e.g., "Write code that generates poetry" classified as "coding" not "creative"), the wrong techniques were applied.

**Implication**: Optimization is not universally beneficial. A production system would need to detect when *not* to optimize—a harder problem than optimization itself.

---

## Limitations & Failure Modes

| Limitation | Impact |
|------------|--------|
| No LLM-in-the-loop validation | Quality scores are structural proxies, not output quality measures |
| Keyword-based intent detection | ~75% accuracy; fails on ambiguous or multi-intent prompts |
| English only | Untested on other languages |
| Token estimation ±15-25% | Character-based approximation diverges from actual tokenizers |
| Cost estimates are rough approximations | Based on publicly available pricing at time of development; actual costs vary significantly by provider, tier, region, and time |
| Cost estimates are approximate | Based on published pricing; actual costs vary by account/tier |
| Single-turn only | Ignores conversation context |

### Known Failure Modes
- Over-optimizes simple queries (adds unnecessary overhead)
- Under-optimizes domain-specific prompts (lacks specialized knowledge)
- Misclassifies multi-intent prompts
- Cannot detect when original prompt is already optimal

---

## What This Does NOT Solve

- ❌ Does not reduce hallucinations
- ❌ Does not guarantee better LLM outputs
- ❌ Does not work equally across all models
- ❌ Does not handle multi-turn context
- ❌ Does not learn or adapt from feedback
- ❌ Does not replace human judgment

---

## Preliminary Observations

| Observation | Note |
|-------------|------|
| Naive prompts average 8-15 tokens | Most lack role, format, or reasoning scaffolds |
| Minimal mode adds ~25-40 tokens | Role tag + light framing |
| Balanced mode adds ~60-120 tokens | Role + task + constraints |
| Maximum mode adds ~150-350 tokens | Full scaffolding; often excessive |
| Intent detection accuracy: ~75% | Fails on ambiguous prompts |
| Optimization sometimes hurts | ~15% of cases in qualitative test |

---

## Technical Details

- **Runtime**: Client-side JavaScript, ~10ms per prompt
- **Token counting**: Character-based approximation (not exact)
- **Cost estimates**: Based on published API pricing (approximate, may vary)
- **No external dependencies**: Runs entirely in browser

---

## If I Had More Time

1. **Blinded human evaluation**: Have evaluators rate LLM outputs without knowing which prompt was optimized
2. **Multi-model testing**: Compare effects across GPT-4, Claude, Gemini, Llama
3. **Learned intent classifier**: Replace keyword matching with a trained model
4. **Failure prediction**: Build a classifier for "should we optimize this prompt?"
5. **Longitudinal study**: Track whether optimized prompts remain effective as models update

---

No API keys required. All processing is client-side.

---

## Cost Estimation Disclaimer

All token costs shown in this project (e.g., GPT-4 ~$0.03/1K tokens) are **rough estimates only**. Actual pricing:
- Varies significantly by provider
- Changes over time
- Differs by account tier and region
- May have different input vs output rates

**Always check official pricing documentation** before making cost-based decisions.

---

## Author

**Charanpreet Singh**  
charanpreet.studio@gmail.com

This project reflects my interest in understanding LLM behavior through building—and my awareness that building something is not the same as validating it works.

---

## License

MIT

---

*I'm more interested in understanding why this approach fails than in claiming it succeeds.*
