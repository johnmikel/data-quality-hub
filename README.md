# Data-Quality Hub

Seven on-device tools for data, schemas and agents — most of them powered by sentence embeddings to do what regex and rule-based validators can't: infer what a schema's fields *mean*, measure *semantic* drift, tell a renamed field from a dropped one, group a list by meaning, trim a prompt to a token budget, redact PII, and score model outputs.

**▶ Live demo: https://www.johnmikelregida.com/labs/ai**

Runs entirely in your browser. No API key, no backend, no data leaves the page — models are downloaded once from a CDN and cached, and all inference happens on your machine.

## What it is

Data contracts break in ways that schemas alone don't catch: a field called `addr` and one called `address` are the same thing; a column can drift in meaning without changing type; a removed field is sometimes a rename in disguise. This page packages seven small tools across data quality, knowledge structuring, LLM tooling and privacy, sharing a single embedding-model load so the cost is paid once. It's aimed at anyone who owns a data contract, a schema registry, an ingestion pipeline, or an LLM/agent workflow and wants a fast, local second opinion before shipping.

It is a single self-contained `index.html`. The embedding model is a quantised 22M-parameter sentence transformer running in WASM — small and fast, not a frontier LLM — so treat its inferences as signal, not ground truth.

## How it works

The four embedding tools share one model, loaded lazily on first run:

- **Library:** `@xenova/transformers@2.17.2` (transformers.js), imported as an ES module from jsDelivr.
- **Model:** `Xenova/all-MiniLM-L6-v2` — mean-pooled, L2-normalised 384-dim embeddings, batched at 48 inputs per call. Because vectors are normalised, cosine similarity is a plain dot product.

**1. Schema Whisperer** — paste a JSON array of records; it derives per field: type (via value regexes), present %, distinct count, an `enum?` flag when distinct ≤ 8, and an example. The interesting part is the *inferred meaning*: it embeds up to 10 string sample values per field, takes their centroid, and compares it against 16 fixed concept anchors (person name, postcode, money, coordinate, …). A label shows only when cosine to the best anchor exceeds **0.44**. It also flags duplicate/synonym field pairs by embedding the field *names* — reported when name cosine ≥ **0.55** OR one name is a lexical prefix of the other — which catches `addr` ↔ `address`.

**2. Drift Lens** — give it two snapshots of a column. It embeds both sets, computes each centroid, and reports drift as `1 − cosine(centroidA, centroidB)`. It lists values that *emerged* in B and *faded* from A (max cross-similarity < **0.55**), then projects all points into 2D with **classical multidimensional scaling** (double-centred similarity matrix, top-2 eigenvectors via power iteration with deflation), coloured by snapshot.

**3. Breaking-Change Radar** — a deterministic schema diff over two JSON Schemas (or sample records). Removed and new-required fields are breaking; new optional fields are safe; type changes use a widening table; enum removal and new enum constraints are breaking. The embedding layer adds **rename detection**: when both sides have orphan fields, it embeds the names and reports a likely rename where the best cross-match has cosine ≥ **0.6** — flagged as "rename + alias, don't drop silently" rather than an unrelated drop + add.

**4. Taxonomy Builder** — give it a flat list, with or without categories. *With* categories, it zero-shot assigns each item to the nearest category by cosine, flagging "needs new category" below **0.30**. *Without*, it auto-discovers groups with **leader clustering** — each item joins the nearest cluster *centroid* above **0.42**, else seeds a new cluster (comparing to the centroid, not any single member, avoids single-linkage chaining where a bridge word merges two groups). Each group is named by its most central member (medoid).

**5. Context Compressor** — paste a long prompt and a token budget. It counts real tokens with **`gpt-tokenizer`** (cl100k, with a word-based fallback), splits the text into sentences, embeds them, and compresses in two passes: first it drops near-duplicate sentences (cosine ≥ **0.78** to one already kept), then, if still over budget, it removes the chunks most redundant with the running centroid (lowest marginal information) until it fits. It shows tokens in/out, reduction %, an illustrative cost delta, and which chunks were cut and why.

**6. PII Scrubber** — detects and redacts personal data entirely on-device. A deterministic regex pass catches emails, phone numbers, credit-card numbers, UK postcodes, UK National Insurance numbers, IPs and URLs. An optional button loads a NER model (`Xenova/bert-base-NER`, token-classification) to additionally redact person names (best-effort, subword-merged). Output is the redacted text plus a per-type breakdown.

**7. Eval Studio** — paste test cases as `{"case","reference","output"}`. It embeds references and outputs and scores each by cosine similarity, passing above a slider threshold, with an aggregate pass rate and average. It's a fast proxy for an LLM-as-judge — it rewards meaning overlap, not factual correctness, and says so — useful for regression-checking prompt or model changes.

```
text / JSON / schema
   ├─ all-MiniLM-L6-v2 (transformers.js, WASM, on-device)
   │     ├─ Schema Whisperer  : value-centroid vs concept anchors + name dedup
   │     ├─ Drift Lens        : centroid cosine drift + emerged/faded + classical MDS map
   │     ├─ Breaking-Change Radar : deterministic diff + embedding rename detection
   │     ├─ Taxonomy Builder  : zero-shot assign OR leader clustering + medoid naming
   │     ├─ Context Compressor: + gpt-tokenizer; dedup + budget-aware redundancy cut
   │     └─ Eval Studio       : reference↔output cosine scoring
   └─ bert-base-NER (optional) : person-name redaction for PII Scrubber
            PII Scrubber also runs a deterministic regex pass (no model)
```

## Why it matters

Schema-as-contract, drift detection and breaking-change gating are core data-platform concerns — what CI checks on a schema registry and what a data SLA depends on. The novel bit is using cheap, local semantic similarity to make those checks *aware of meaning*: synonym columns, drifting categoricals and renames are exactly the failures that purely structural diffs miss. Taxonomy, compression, PII and eval extend the same idea into knowledge structuring, LLM-cost control, privacy and evaluation. And the underlying pattern — small models running client-side with zero data egress — is increasingly relevant to ML-platform and agent work where you want semantic matching without shipping records to an API.

## Run it locally

These are static files that fetch ES modules and WASM from CDNs at runtime, so they must be served over HTTP — opening `index.html` via `file://` will fail on module/CORS loading.

```bash
cd ai-hub
python3 -m http.server 8000
# then open http://localhost:8000/index.html
```

The first embedding run downloads the model (~25 MB) and caches it in the browser; the optional NER model and `gpt-tokenizer` are fetched on demand the first time you use PII names / Context Compressor. There is no build step and no server — the entire app is the one HTML file.

## Tech

- transformers.js (`@xenova/transformers@2.17.2`): `Xenova/all-MiniLM-L6-v2` embeddings + optional `Xenova/bert-base-NER` token-classification, WASM, on-device.
- `gpt-tokenizer` (cl100k) for exact token counts, loaded on demand with a heuristic fallback.
- Vanilla JS + inline CSS, single self-contained `index.html`, no framework, no build.
- From-scratch numerics: cosine via dot product on normalised vectors, centroids, classical MDS (power iteration + deflation), leader clustering, medoids.
- Demo presets are illustrative sample data, not live datasets.

---

Built by John Mikel Regida — Lead Data Architect (Thoughtworks; UK Dept for Transport / NaPTAN; ex-CTO; 5× Google Cloud Professional). GitHub: github.com/johnmikel. Site: https://www.johnmikelregida.com

Part of the JMR Labs suite — https://www.johnmikelregida.com/labs
