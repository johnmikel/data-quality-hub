# Data-Quality Hub

Three on-device data-quality tools that use sentence embeddings to do what regex and rule-based validators can't: infer what a schema's fields *mean*, measure *semantic* drift between data snapshots, and tell a renamed field apart from a dropped one.

**▶ Live demo: https://www.johnmikelregida.com/labs/ai**

Runs entirely in your browser. No API key, no backend, no data leaves the page — the embedding model is downloaded once from a CDN and cached, and all inference happens on your machine.

## What it is

Data contracts break in ways that schemas alone don't catch: a field called `addr` and one called `address` are the same thing; a column of values can drift in meaning without changing type; a removed field is sometimes a rename in disguise. This page packages three small tools that address those gaps, sharing a single embedding-model load so the cost is paid once. It's aimed at anyone who owns a data contract, a schema registry, or an ingestion pipeline and wants a fast, local second opinion before shipping a change.

It is a single self-contained `index.html`. The model is a quantised 22M-parameter sentence transformer running in WASM — small and fast, not a frontier LLM — so treat its inferences as signal, not ground truth.

## How it works

All three tools share one model, loaded lazily on first run:

- **Library:** `@xenova/transformers@2.17.2` (transformers.js), imported as an ES module from jsDelivr.
- **Model:** `Xenova/all-MiniLM-L6-v2` — mean-pooled, L2-normalised 384-dim embeddings, batched at 48 inputs per call. Because vectors are normalised, cosine similarity is a plain dot product.

**1. Schema Whisperer** — paste a JSON array of records; it derives per field: type (via value regexes — email, url, ISO date, integer, number, etc.), present %, distinct count, an `enum?` flag when distinct ≤ 8, and an example. The interesting part is the *inferred meaning*: it embeds up to 10 string sample values per field, takes their centroid, and compares it against 16 fixed concept anchors (person name, postcode, money, coordinate, identifier, free text, …). A label is shown only when cosine to the best anchor exceeds **0.44**. It also flags likely duplicate/synonym field pairs by embedding the field *names* — a pair is reported when name cosine ≥ **0.55** OR one name is a lexical prefix of the other (≥ 3 chars), which catches `addr` ↔ `address`.

**2. Drift Lens** — give it two snapshots of a column (one value per line). It embeds both sets, computes each snapshot's centroid, and reports drift as `1 − cosine(centroidA, centroidB)`. It lists values that *emerged* in B (max similarity to any A value < **0.55**) and values that *faded* from A (symmetric). It then projects all points into 2D with **classical multidimensional scaling** — double-centring the similarity matrix and extracting the top two eigenvectors via power iteration with deflation — and plots them coloured by snapshot.

**3. Breaking-Change Radar** — a deterministic schema diff over two JSON Schemas (or two sample records, which it normalises). It classifies each change as breaking or safe: removed fields and new *required* fields are breaking; new optional fields are safe; type changes use a widening table (`integer → number | string`, `number → string` are safe, everything else is narrowing/breaking); enum value removal and newly-added enum constraints are breaking; requiredness tightening is breaking, relaxing is safe. The embedding layer adds **rename detection**: when both sides have orphan fields, it embeds the removed and added names and, where the best cross-match has cosine ≥ **0.6**, reports it as a likely rename (still breaking for consumers, but flagged as "rename + alias, don't drop silently") instead of an unrelated drop + add.

```
JSON / column text
   └─► all-MiniLM-L6-v2 (transformers.js, WASM, on-device)
          ├─ Schema Whisperer  : value-centroid vs concept anchors  + name-similarity dedup
          ├─ Drift Lens        : centroid cosine drift  + emerged/faded + classical MDS map
          └─ Breaking-Change Radar : deterministic diff + embedding rename detection
```

## Why it matters

Schema-as-contract, drift detection, and breaking-change gating are core data-platform concerns — they're what CI checks on a schema registry and what a data SLA depends on. The novel bit here is using cheap, local semantic similarity to make those checks *aware of meaning*: synonym columns, drifting categoricals, and renames are exactly the failures that purely structural diffs miss. The same on-device-inference pattern — a small embedding model running client-side with zero data egress — is increasingly relevant to ML-platform and agent work where you want semantic matching without shipping records to an API.

## Run it locally

These are static files that fetch ES modules and WASM from CDNs at runtime, so they must be served over HTTP — opening `index.html` via `file://` will fail on module/CORS loading.

```bash
cd ai-hub
python3 -m http.server 8000
# then open http://localhost:8000/index.html
```

The first analysis downloads the model (~25 MB) and caches it in the browser; subsequent runs are instant and offline. There is no build step, no data pipeline, and no server — the entire app is the one HTML file.

## Tech

- transformers.js (`@xenova/transformers@2.17.2`), `Xenova/all-MiniLM-L6-v2` sentence embeddings, WASM, on-device.
- Vanilla JS + inline CSS, single self-contained `index.html`, no framework, no build.
- From-scratch numerics: cosine via dot product on normalised vectors, centroid computation, classical MDS via power iteration with deflation.
- Demo presets are UK public-transport flavoured (NaPTAN/ATCO-style records, transit-mode vocabularies) and are illustrative sample data, not a live dataset.

---

Built by John Mikel Regida — Lead Data Architect (Thoughtworks; UK Dept for Transport / NaPTAN; ex-CTO; 5× Google Cloud Professional). GitHub: github.com/johnmikel. Site: https://www.johnmikelregida.com

Part of the JMR Labs suite — https://www.johnmikelregida.com/labs
