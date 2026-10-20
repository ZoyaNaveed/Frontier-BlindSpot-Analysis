# Schema-Forced Fabrication: when a required field has no answer, small frontier models invent one

**Model evaluated:** [Qwen/Qwen3.5-4B](https://huggingface.co/Qwen/Qwen3.5-4B) (thinking on and off)
**Author:** Zoya Naveed
**Artifacts in this bucket:** evaluation dataset, prompts/schemas, raw model outputs, scored results, figures, notebook.

> TL;DR — _[Fill after running: e.g. "Qwen3.5-4B extracts present HPI fields with X% accuracy, yet invents a value for Y% of fields the patient never mentioned. Allowing `null` or an explicit `not_mentioned` reduces this to Z%, and adding realistic 'complete the record' pressure pushes it back up to W%. Thinking mode changes it by …"]_

---

## 1. The blind spot (Question 1)

### What I saw in practice
_[Write this in your own voice — this is the part graders weigh most. Points to cover:]_
- _You build a conversational patient-intake system at a healthcare company that turns patient conversations into structured HPI records (onset, location, duration, character, severity, aggravating/relieving factors, associated symptoms, medications)._
- _Downstream, these records are materialized into structured clinical formats (form components / FHIR resources) where fields are typed and often required._
- _A concrete moment you noticed the model filling a field the patient never talked about (e.g. a severity score, or "no other symptoms") and why it was hard to catch — the output looked clinically plausible and schema-valid._

### Why this matters
- A fabricated value is **indistinguishable from a real one** once it is in the record. A missing value is visible and can be asked about; an invented "5/10" cannot.
- Fabricated *negatives* ("no associated symptoms", "no medications tried") are arguably worse: they close questions a clinician would otherwise ask.
- Production pipelines enforce schemas (JSON mode, constrained decoding, EHR validation), which structurally **reward completeness over truthfulness**.

### Why standard benchmarks miss it
- Extraction benchmarks score fields whose answers *exist* in the source. Missing-field behavior is rarely in the denominator.
- Qwen3.5-4B scores 82.4 mean on ExtractBench (structured-output extraction) per its model card — a strong extractor by benchmark standards. This evaluation asks the complementary question: what does it do when there is nothing to extract?
- Hallucination benchmarks focus on free text and world knowledge, not on schema-induced fabrication of *patient-specific* facts.

---

## 2. Why this model

- _Current frontier-lab family (Qwen3.5, 2026), heavily deployed (~7.8M monthly downloads), Apache-2.0._
- _Hybrid Gated DeltaNet + attention architecture representative of current efficient frontier designs; thinking/non-thinking toggle lets me test whether reasoning mitigates the gap._
- _4B fits a free Colab T4, which is also realistic for on-prem healthcare deployment where PHI cannot leave the network._

---

## 3. Evaluation design

**Minimal pairs.** Each synthetic vignette is a doctor–patient intake transcript built from one turn per HPI field, with known ground truth per field. For each vignette, *k* ∈ {0, 2, 4, 6} field turns are removed (same seed across all conditions), so the only thing that changes between paired items is whether the information exists.

| Axis | Levels |
|---|---|
| Missing fields (k) | 0 (control), 2, 4, 6 |
| Schema / instruction | `strict` (all required, no absence option), `nullable`, `sentinel` (`"not_mentioned"` + "do not guess"), `sentinel_pressure` (same + "incomplete records are rejected by the EHR") |
| Reasoning | thinking off (all), thinking on (subset: k=4, strict & sentinel) |

- **Dataset:** _N_ vignettes × 4 k-levels = _M_ items; all synthetic, no real patient data. Fields judged trivially inferable from the chief complaint (e.g. "location" for a knee complaint) are never removed.
- **Decoding:** greedy, fixed prompts (see `dataset/conditions.json`).
- **Scoring (absent fields):** `abstain` (null / not_mentioned / "unknown"…), `fabricated_negative` (asserts absence, e.g. "none"), `fabricated_positive` (invents a value).
- **Scoring (present fields):** correct / wrong / false abstention (fuzzy match; spot-checked by hand).
- **Uncertainty:** 95% CIs via bootstrap over vignettes.
- _[Optional: clinician spot-check of N vignettes — say so if you get one.]_

---

## 4. Results

_[Insert figures from `figures/` and the table from `results/summary.csv`.]_

1. **Headline** — `figures/headline.png`: present-field accuracy vs. absent-field fabrication per condition.
2. **Dose-response** — `figures/by_k.png`: fabrication as more information is missing.
3. **Which fields** — `figures/by_field.png`: _[e.g. severity and relieving factors are fabricated most; these have strong population priors.]_
4. **What the fabrications look like** — `figures/fabricated_severity.png`: _[e.g. invented severities cluster at 5–6 — values come from priors, not the patient.]_
5. **Thinking mode** — `figures/thinking.png`: _[does reasoning help, or does it rationalize the guess?]_

### Failure examples
_[4–6 short excerpts: transcript snippet → absent field → model output → why it matters clinically.]_

### Story
_[3–5 sentences connecting the findings: competent extractor; abstention is instruction-sensitive but fragile; production-style pressure reverses it; fabricated values are prior-driven; negatives are a distinct and dangerous sub-type.]_

---

## 5. Path forward (Question 3)

_[Edit into your own words; keep the ones your results support most.]_

**Data curation — counterfactual deletion pairs.** Take existing grounded extraction examples and create paired copies with the supporting span deleted and the gold label set to an explicit "not mentioned". The model then sees the same schema with and without evidence, so it learns that the schema does not license a value — evidence does. Weight toward high-prior fields (severity, negatives) where fabrication concentrates.

**Fine-tuning — preference optimization against the model's own fabrications.** Sample the model's fabricated outputs on absent fields (on-policy negatives) and train with DPO: *chosen* = abstention, *rejected* = plausible fabrication, including under "complete the record" pressure prompts so abstention survives operational pressure.

**RL with a verifiable grounding reward.** Require each non-null value to cite a supporting span from the transcript. Whether the span exists is checkable by string match, so a reward of "value correct, or abstained when no span exists; penalty for an un-grounded value" is cheap and fully automatic — no human labels needed. This directly targets the incentive that likely causes the gap: post-training rewards helpful-looking completeness.

**Architecture / inference.**
- Make absence first-class in structured decoding: every field's grammar includes an "absent" branch, and the schema pairs each value with an `evidence` span that a post-hoc verifier checks before the record is committed.
- Two-stage extract-then-verify: a second pass asks only "is this value stated in the transcript?" per field — a narrower question small models answer more reliably than open extraction.
- Calibration signal: use per-field token probabilities of the "absent" branch vs. a value to route low-confidence fields to a human or a follow-up question in the conversation.

**Evaluation practice.** Extraction benchmarks should report a *missing-field* metric alongside accuracy; a model that fills everything should not score as the best extractor.

---

## 6. Limitations
- Synthetic, English-only transcripts with one clean turn per field; real intake is messier (code-mixed speech, partial answers). _[Mention Roman Urdu / code-mixed as future work if you didn't run it.]_
- Fuzzy matching for present fields is approximate (spot-checked).
- Greedy decoding only; single model family (plus _[second model if run]_).
- Some removed information may be weakly inferable from other turns; those cases count as fabrication here because nothing was stated.

---

## 7. Bucket contents & reproduction

```
dataset/vignettes.jsonl       # source vignettes with per-field turns and gold values
dataset/eval_items.jsonl      # rendered transcripts, removed fields, gold per item
dataset/conditions.json       # system prompt, instructions and JSON schemas per condition
results/raw_outputs.jsonl     # every model generation
results/scored_fields.csv     # one row per item × condition × field with outcome
results/summary.csv           # aggregated rates with bootstrap CIs
results/run_config.json       # model, dtype, decoding, GPU
figures/*.png
schema_fabrication_eval.ipynb # end-to-end notebook (Colab T4)
```

Base model weights: [Qwen/Qwen3.5-4B](https://huggingface.co/Qwen/Qwen3.5-4B) _[or `model/` if copied into this bucket]_.
