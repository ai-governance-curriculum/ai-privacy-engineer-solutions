# Exercise 04 — LLM Memorisation and Extraction Evaluation — Reference Solution

## 1. Solution overview

This is an **implementation + reporting** exercise. The learner builds an
evaluation that distinguishes two different things LLM stakeholders
routinely conflate:

- **Verbatim memorisation (discoverable extraction):** given a prompt that
  is a prefix of a training example, can the model emit the exact
  continuation? Canonical reference: Carlini, Ippolito, Jagielski, Lee,
  Tramèr, Zhang, "Quantifying Memorization Across Neural Language Models,"
  ICLR 2023.
- **Untargeted extraction (extractable memorisation):** given *no*
  knowledge of the training set, can the attacker elicit verbatim training
  text from the model? Canonical reference: Carlini, Tramèr, Wallace, et al.
  "Extracting Training Data from Large Language Models," USENIX Security
  2021; and the scaled-up "Scalable Extraction of Training Data from
  (Production) Language Models," Nasr, Carlini, et al. 2023.

Both map to OWASP Machine Learning Security Top 10 **ML02 Data Poisoning
Attack** (the inverse direction: training-data traces re-surface at
inference) and **ML03 Model Inversion / Membership Inference**, and to
MITRE ATLAS **AML.T0024 Exfiltration via ML Inference API**
(<https://atlas.mitre.org/>). The deliverable is a two-pronged measurement
harness and a report.

Deliverables:

1. A **discoverable memorisation** evaluation: given a corpus the learner
   knows is in training (or a synthetic corpus the learner *can* put in
   training), measure the rate at which the model emits the exact
   continuation as a function of prompt length `P`, model size, and
   example repetition count `R`.
2. An **extractable memorisation** evaluation: without training-set
   knowledge, drive the model with a prompting / sampling strategy, filter
   candidates with a membership oracle (trigram index, online search,
   known-corpus lookup), and count unique verbatim training snippets
   emitted per unit of compute.
3. A report that reads the two numbers together and routes to downstream
   mitigations.

## 2. Worked implementation

### 2.1 Metrics the learner must emit

- **k-extractable (Carlini et al. 2023 Def. 1 adaptation):** a string
  `s = [p, c]` of length `|p| + |c|` is *k-extractable* from model `f` if,
  given prompt `p` of length `|p| ≥ k`, greedy decoding of `f` emits
  exactly `c`. Report the fraction of eval strings that are k-extractable
  as a function of `k` and `|c|`.
- **Memorisation rate curve:** fraction extractable vs example repetition
  count `R` (requires controlled training; see §2.4). Carlini et al. 2023
  shows this curve rises super-linearly beyond a small `R`.
- **Extraction yield:** unique verbatim training snippets recovered per
  `10^6` sampled tokens under a stated sampling strategy. Report per
  prompt family (web, source code, structured records, URLs), per model
  size.
- **PII hit rate:** fraction of extracted snippets that contain any of a
  pre-committed list of pattern types (email, phone, name+address,
  credit-card-shaped). Use Microsoft Presidio
  (<https://microsoft.github.io/presidio/>) or the equivalent detector
  from mod-104. The hit rate is a *downstream* risk signal.

### 2.2 Discoverable memorisation — reference pseudocode

```python
# discoverable.py
from typing import List

def is_k_extractable(model, prefix_tokens, suffix_tokens, max_new):
    """Greedy continuation equals the suffix exactly."""
    out = model.generate(prefix_tokens, do_sample=False,
                         max_new_tokens=max_new, temperature=0.0)
    gen = out[len(prefix_tokens):len(prefix_tokens) + len(suffix_tokens)]
    return gen == suffix_tokens

def discoverable_rate(model, eval_examples, k_values):
    """eval_examples: List[List[int]] — tokenised training-set strings."""
    rates = {}
    for k in k_values:
        hits = 0; total = 0
        for toks in eval_examples:
            if len(toks) <= k + 1: continue
            prefix, suffix = toks[:k], toks[k:k + 50]  # measure 50-token continuation
            total += 1
            if is_k_extractable(model, prefix, suffix, max_new=50):
                hits += 1
        rates[k] = hits / max(total, 1)
    return rates
```

Report as a table `k ∈ {25, 50, 100, 200}` vs model size vs rate. On a
log-log plot you should see roughly the trend reported in Carlini et al.
2023 §4 — memorisation rises with both prefix length and model scale on the
same training set. The learner's own numbers may differ; *that is the
point* of the exercise. Do not restate the paper's numbers as if they were
the learner's.

### 2.3 Extractable memorisation — reference pseudocode

```python
# extractable.py
import random
from collections import Counter

PROMPT_STRATEGIES = {
    "web_prefix":     lambda rng: random.choice(WEB_SEEDS) + " ",
    "code_prefix":    lambda rng: random.choice(CODE_SEEDS) + "\n",
    "structured":     lambda rng: random.choice(STRUCT_SEEDS),
    "divergence":     lambda rng: "repeat the word '" + random.choice(WORDS) + "' forever:\n",
    # The last strategy is the Nasr et al. 2023 "divergence" prompt family;
    # include it as one option — not as the whole experiment.
}

def extract_round(model, strategy_name, n_samples, tokens_per_sample, membership_oracle):
    """Returns Counter of (snippet -> occurrences) for snippets confirmed as
    training-set members by the oracle."""
    found = Counter()
    for _ in range(n_samples):
        prompt = PROMPT_STRATEGIES[strategy_name](random)
        out = model.generate(prompt, do_sample=True, top_p=0.95,
                             max_new_tokens=tokens_per_sample)
        # Candidate = any 50-token window with >5 non-whitespace tokens
        for window in sliding_windows(out, size=50, stride=25):
            if membership_oracle(window):
                found[window] += 1
    return found
```

The **membership oracle** is the critical design choice. Options, in order
of preference for the classroom version:

1. **Training-set hash index.** When the learner controls the training
   corpus (classroom fine-tune), build a suffix-array or n-gram index
   over it and query exactly.
2. **Deduplicated web corpus proxy.** For a pre-trained model, a large
   dedup'd web corpus (e.g., Common Crawl slice) is a weak oracle for "is
   this likely training data." Report all such findings as *suspected*
   extractions, not confirmed.
3. **Online text-search.** Only legitimate for prompts emitting
   public-domain-looking text; it overcounts common boilerplate.

### 2.4 Experimental setup for the memorisation-rate curve

To get a clean memorisation-rate-vs-repetition curve, the learner needs
*controlled* training. Two safe options for the classroom:

- Fine-tune a small open-weights model on a synthetic corpus of ~10 000
  short, distinctive, **non-PII** strings (e.g., algorithmically generated
  pangrams with a seed, or random technical descriptions of fictitious
  objects) where each string is duplicated R ∈ {1, 4, 16, 64, 256} times.
- Then evaluate discoverable extraction per R bucket.

Generating distinctive **non-PII** training strings is a required safety
property of this exercise — never inject real PII or copyrighted content
for the sake of an extraction demo.

### 2.5 Report structure (required)

Two pages. The reviewer must be able to answer four questions:

1. **How bad is discoverable memorisation at the prefix lengths the
   product actually exposes?** Table: k vs rate.
2. **How bad is extractable memorisation per unit of attacker
   compute?** Table: strategy × samples × yield.
3. **What fraction of extracted content contained PII patterns?** Hit
   rate per pattern family. Zero PII hits is not proof of no risk — the
   patterns only find the shapes you looked for.
4. **What did we not test?** Non-English, multimodal, long-horizon
   decoding, adversarial system-prompt bypass, agent / tool-use loops.
   State the capability gap.

### 2.6 Handover to the rest of the module

- High discoverable rate at short `k` → route to mod-106 (DP-SGD on
  fine-tunes) and to dataset deduplication (mod-103).
- Non-zero extractable PII hit rate → route to mod-104 (detection /
  redaction in training data) and open a DSR impact review (mod-110).
- Report an **epsilon claim if DP was used** or state that no formal
  guarantee applies.

## 3. Validation steps

1. **Discoverable sanity.** Insert a seeded, distinctive non-PII string
   into fine-tune data with repetition R=256; after training, the model
   should emit the continuation on a short prefix. If it does not, the
   training run is too short or the data pipeline dropped the duplicates.
2. **Extractable sanity.** On an untrained model (randomly initialised),
   the extraction harness should produce **zero** oracle hits. If it does
   not, the oracle is too permissive and is matching on common n-grams.
3. **PII hit-rate sanity.** Run the detector on a known PII corpus and
   verify the detector's own TPR/FPR for your pattern families. The
   detector's error rate caps the measurement's error.
4. **De-duplicated oracle.** Deduplicate your training corpus before you
   build the membership oracle. Carlini et al. (ICLR 2023) and Lee et al.
   "Deduplicating Training Data Makes Language Models Better," ACL 2022
   both show duplicates drive memorisation; the oracle must reflect the
   effective training distribution, not the raw corpus.
5. **Decoding setting matters.** Report greedy, nucleus (`top_p=0.95`),
   and temperature=1.0 separately. The Nasr et al. 2023 result that
   production models can be made to emit training data under a specific
   sampling / prompting regime does not transfer to greedy decoding.

## 4. Rubric or review checklist

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| k-extractable rate reported for at least three prefix lengths | 2 | Including a "short enough to match product UX" value |
| Memorisation-rate vs repetition curve from controlled training | 2 | Fine-tune or small from-scratch |
| Extractable-memorisation yield reported per attacker-compute unit | 2 | Strategy × samples × hits |
| Membership oracle described and its FPR acknowledged | 1 | Named corpus or hash index |
| PII hit-rate on extracted snippets | 1 | Zero-hits is not proof of no risk |
| Explicit "capability gap" paragraph | 1 | Lists what was not tested |
| No real PII or copyrighted text injected to produce a demo | 1 | Automatic fail if violated |

Automatic failures:

- Injecting real user data, real PII, or copyrighted text into training to
  manufacture an extraction demo. Use synthetic, seeded, non-PII strings.
- Reporting "zero memorisation" from greedy decoding and generalising to
  product risk under unknown decoding regimes.
- No membership oracle described.

## 5. Common mistakes

- **Mixing discoverable and extractable numbers.** Discoverable rate
  measures what the model *can* reproduce when you *know* the prefix.
  Extractable yield measures what an attacker *can* harvest without that
  knowledge. They are not directly comparable and do not aggregate.
- **Running the Nasr et al. 2023 "divergence" prompt and nothing else.**
  It is one prompt family among many; treating it as *the* extraction
  benchmark biases the result.
- **Using a weak membership oracle.** An n-gram match against unduplicated
  Common Crawl is high-FPR. Report candidates and confirmed extractions
  separately.
- **Testing in English only.** Multilingual LLMs have per-language
  memorisation curves; the report's capability-gap paragraph exists to
  prevent silent monolingual claims.
- **Confusing rote memorisation with copyright-relevant copying.** A
  model that reproduces a short common phrase is not the same as one that
  reproduces a specific author's paragraph. The report must not conflate
  "any verbatim match" with "regulated disclosure."
- **Running once with one seed.** Memorisation measurements have
  non-trivial variance; report at least three seeds for sampling-based
  metrics and a confidence interval.

## 6. References

- Nicholas Carlini, Daphne Ippolito, Matthew Jagielski, Katherine Lee,
  Florian Tramèr, Chiyuan Zhang. "Quantifying Memorization Across Neural
  Language Models." ICLR 2023. <https://arxiv.org/abs/2202.07646>
- Nicholas Carlini, Florian Tramèr, Eric Wallace, Matthew Jagielski, Ariel
  Herbert-Voss, et al. "Extracting Training Data from Large Language
  Models." USENIX Security 2021. <https://arxiv.org/abs/2012.07805>
- Milad Nasr, Nicholas Carlini, Jon Hayase, Matthew Jagielski, A. Feder
  Cooper, Daphne Ippolito, Christopher A. Choquette-Choo, Eric Wallace,
  Florian Tramèr, Katherine Lee. "Scalable Extraction of Training Data
  from (Production) Language Models." 2023.
  <https://arxiv.org/abs/2311.17035>
- Katherine Lee, Daphne Ippolito, Andrew Nystrom, Chiyuan Zhang, Douglas
  Eck, Chris Callison-Burch, Nicholas Carlini. "Deduplicating Training
  Data Makes Language Models Better." ACL 2022.
  <https://arxiv.org/abs/2107.06499>
- OWASP Machine Learning Security Top 10.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS, AML.T0024 Exfiltration via ML Inference API.
  <https://atlas.mitre.org/techniques/AML.T0024>
- NIST AI Risk Management Framework (AI RMF 1.0), GOVERN & MEASURE.
  <https://www.nist.gov/itl/ai-risk-management-framework>
- Microsoft Presidio. <https://microsoft.github.io/presidio/> — used as
  the PII detector in the hit-rate step; see mod-104 for the detection
  pipeline.
