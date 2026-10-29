# Armore — The Compounding Moat

### How to make Armore self-improving, so that any copy falls further behind every month

| | |
|---|---|
| **Status** | Proposal for operator review — 2026-10-29 |
| **Prepared for** | Armore founders and engineering leads |
| **Evidence base** | `armore-edge-internal` @ `d4717336c0` · `armore-edge-mobile` @ `1f17215e08` · `edge-model-engine` @ `0ffe69ce5e` · `armore-verify` @ `e596a684eb` (default branches, read 2026-10-29) |
| **Confidentiality** | **Disclosure-gated.** Sections 5 and 6 describe mechanisms that may be patentable. Per internal's `docs/README.md`, nothing that could forfeit novelty goes public before patent counsel clears it (India/EU absolute novelty). Do not publish, present, or paste into a public issue. |

Claims are tagged the way your own docs tag them: **[measured]** = I read the code or ran a command at the pinned commit · **[doc]** = stated in your docs, not re-verified · **[external]** = checked against a third-party source on 2026-10-29 · **[estimate]** = engineering judgement, a hypothesis to measure.

---

## 0. The answer in one paragraph

Nobody can be stopped from copying an idea, and you plan to make your core copyable on purpose, because open source "IS the trust claim" (`docs/strategy/positioning.md`). No source is public yet: the source flip (#783) waits on patent filings, and ADR-0027 records the licence itself as undecided. But the desktop app's JavaScript already ships, readable, inside every public download. That settles the question: **the moat cannot be the code or the idea.** It has to be what a fork does not get. A competitor who forks Armore on any given day gets a **snapshot** — the code, the open models, that day's quality. Armore keeps the **slope** — a learning system that turns every user correction, every automatic verifier check, every new open-model release and every opted-in fleet measurement into a better, signed, provable model, *without user data ever leaving the user's devices*. If the slope is steep and its inputs stay closed, the fork falls further behind every month and cannot catch up by forking again, because the parts that compound — the hidden eval corpus, the synthetic-data engine, the training recipes, the learned adapters, the routing priors, the opted-in fleet, and the track record of signed proofs — are never in the open code.

**This is not a bet that small models catch up with frontier models.** Your long-game analysis calls that bet "most likely false" and rules that value must be decoupled from model quality: "our value is the sovereignty envelope rather than the weights inside it" (`docs/book/part2/04-the-long-game.md` §4.3). The loop described here is part of that envelope. It is model-agnostic machinery — capture, verifiers, exemplar memory, a hidden eval corpus and a model factory — that makes *whatever* model runs inside the envelope (an open model, an Armore-tuned one, or a customer's own) measurably better on *the user's own documents*, and proves it. The comparison it wins is not Armore vs. GPT. It is Armore this month vs. a fork of Armore.

**Honest status:** that loop has not completed a single turn yet. This document is the design for it, plus the short list of things blocking it.

---

## 1. Executive summary

### 1.1 What you already have that is hard to copy

1. **One Rust core behind every surface.** Desktop (NAPI-RS), CLI, iOS and Android (UniFFI) share one implementation of identity, crypto, capabilities, the sovereignty ceiling and model provisioning [measured: `armore-edge-mobile/rust/armore-core/Cargo.toml` pins internal's `armore-net`, `armore-chain`, `armore-cap`, `armore-compute-router`, `armore-drive`, `armore-release` at one rev].
2. **A verifiable-sovereignty spine that nobody else in local AI has.** Signed execution receipts that say *which brain answered* (Local / Routed / LocalButEgressed), a three-axis egress ledger (where it ran · who produced it · whether content crossed), a Merkle audit export, and an **independent** verifier written in a different language on purpose (`armore-verify/CHARTER.md`).
3. **An eval-honesty culture that is rare and valuable.** "Every quality number ships with the eval that reproduces it" (`edge-model-engine/docs/WORKING-AGREEMENT.md`); hidden-test grading in `docs/strategy/ai-model-scorecard.md`; "every assertion ships with the mutation that makes it fail" (armore-verify Charter §4).
4. **The mesh** — your stated wedge — which is also the ideal substrate for *private* learning: a Pro home node is a training server the user already owns, and your own research names mesh-parallel best-of-N as the top accuracy lever (`docs/strategy/edge-accuracy-research.md`).

### 1.2 What is missing for "self-improving"

1. **The learning engine has stalled.** `edge-model-engine` has 16 commits and nothing merged since 2026-07-23. Its LoRA pipeline has **never completed a run** — the September fix branch (`fix/12-colab-notebook-preflight`) found the notebook "could never have reached Step 3" and is still unmerged [measured]. Since 2026-09-01 alone, internal landed ~1,440 commits: 460 touched `docs/`, 216 touched `.github/workflows/`, 126 touched `scripts/ci/` — and **1** touched `bench/` [measured].
2. **No closed feedback loop in the product** [measured]:
   - Extract output is read-only; its only edit control is the tamper demo (`Extract.tsx:372`, `:597`).
   - No AI output can be rated anywhere.
   - Rejected code hunks are reverted but not logged.
   - `route-jobs.json` records every routed job and nothing reads it back.
   - The LoRA hooks in the runtime (`set_lora` in `crates/drive/src/extraction.rs`, `LocalLlm.setLora` in the SDK) have no caller in the app.

   Nothing the product learns from use flows back into it — the loop is open at both ends.
3. **The eval set is tiny and synthetic:** 30 invoices + 12 receipts, and the scanned sets are synthetic renders (`benchmarks/ocr_scan_eval.md` says so itself). At this size, the 95.4% that lets `qwen2.5:3b` "clear" the 95% bar sits inside a ±2.7 pp confidence interval — the loop has nothing precise to steer by (§7, F3).
4. **Engine and product are disconnected.** The engine speaks Ollama tags and Python; the product speaks a GGUF catalog, llama.cpp and TypeScript/Rust. The only bridge today is a hand-copied number (`extractionAccuracyPct: 95.4` in `packages/app/src/modelCatalog.ts`). Worse, the engine measures **with** constrained decoding, which shipped extraction does not use, so shipped extraction accuracy is unmeasured. And there is no channel to ship a better model without an app release [measured; §7, F4].
5. **A licence landmine (verified today).** The product's default vision/OCR model is Qwen2.5-VL-3B, whose upstream licence is `qwen-research` (non-commercial); your catalog records Apache-2.0 because it read a re-uploader's model card. And the engine's headline recommendation, `qwen2.5:3b`, is a model your own licence guard already blocks. **You cannot build a proprietary learned asset on a non-commercial base.** Details in §7, F1.
6. **The IP clock has run out — check it today.** Your long-game chapter records **2026-10-28** (yesterday) as the absolute drop-dead for filing before India/EU novelty is lost; the latest status in the repo is "not yet filed" (2026-10-07). Filings are held off-repo by design, so I cannot see whether it happened. Separately, the root `LICENSE` file is **MIT** while `Cargo.toml` and the SDK say **Apache-2.0**, and ADR-0027 records the licence as undecided — and the two licences treat patents differently. Details in §7, F0.

### 1.3 The design in one table

Four learning loops at four scopes. The outer loops make the inner ones better. Only proofs and noised aggregates ever cross a device boundary, and only with consent.

| Loop | Scope | Tier | What learns | What leaves the device |
|---|---|---|---|---|
| **L0** | one device | Free | exemplar memory built from your corrections, verifiers, best-of-N, router calibration — **no training** | nothing |
| **L1** | your workspace, on your home node | Pro | a private LoRA adapter, trained at idle on your verified examples | nothing leaves your mesh |
| **L2** | your devices first, then the opted-in fleet | Pro / Team | in-mesh federated evaluation (your FL-1); then, opt-in: eval verdicts, privatised failure taxonomy, router priors; later federated adapters (your patent-pending lane) | in-mesh: nothing · fleet: consent-receipted, noised aggregates |
| **L3** | Armore's model factory | — | base models and adapters from synthetic data, open-weight teachers and new open releases | n/a — zero customer data |

### 1.4 The moat stack, ordered by how long a copier needs

1. **Learning velocity of the envelope** — the compounding asset. It is model-agnostic, so it survives bring-your-own-model (§3.4, §5).
2. **Proof track record** — a signed eval attestation for every model, linked from every inference receipt. Receipt *formats* will become mechanisable by standards; signed *history* will not (§6.3).
3. **Closed learning assets** — hidden eval corpus, synthetic-data engine, recipes, current adapters, router priors, vertical packs (§6.1).
4. **Privacy-preserving network effects** — workspace, fleet, ecosystem (§6.5).
5. **IP** — patents filed before disclosure, trade secrets, trademarks, a clean model-licence chain (§6.4).
6. **Distribution and compliance position** — design partners, regional channels, auditor acceptance (§6.6).

### 1.5 The next 30 days

0. **Today: confirm patent filing status with counsel**, and settle MIT vs Apache-2.0 before any source flip (§7, F0).
1. **Fix licence lineage.** Move the default VLM to an upstream-permissive model (Qwen2.5-VL-7B is Apache-2.0 upstream [external]); resolve every catalog licence from the *upstream* card via `base_model`; make `recommend()` licence-aware.
2. **Turn the flywheel once.** Run the fine-tune as a scripted job on a rented GPU (not a manual Colab session) and publish adapter v1 with a signed eval report on held-out data.
3. **Ship correction capture + exemplar memory in Extract.** Inline field edit → signed local learning event → your nearest corrected documents retrieved as few-shot examples.
4. **Write the open/closed boundary for learned assets** into the OSS export manifest before anything else is published.
5. **Brief patent counsel** on §5–§6 before any of it lands in open-licensed code.

---

## 2. What I analysed

| Repo | What it is | Size & activity | What matters for this strategy |
|---|---|---|---|
| `armore-edge-internal` | The product: Iroh mesh, signed Automerge sync, capability tokens, Merkle audit, on-device AI (llama.cpp, Tesseract, VLM), Electron app, CLI, TS SDK | 3,080 files (1,100 TS + 273 TSX, 340 Rust), 494 markdown docs (~8 MB); 4,967 commits since 2026-06-01, 881 in October | AI features ship app-side; model catalog + licence guard; device-fit routing; scorecard; distillation bench; FL kept out by `novel-signature-guard.yml` |
| `armore-edge-mobile` | Native SwiftUI + Compose over the shared Rust core (UniFFI) | 735 commits since 2026-07-16; 22 in October | Digest-verified model provisioning with per-purpose active slots (`provision.rs`); `ExtractionEngine` export exists but is feature-gated; on-device inference still gated on the unruled #233 scope decision (`docs/mobile-ai-cost-memo.md`) |
| `edge-model-engine` | "Find — and prove — the smallest open model that clears a quality bar" | ~3.3k lines of Python, 16 commits, idle since 2026-07-23 | Eval harness, model × quant sweeps, constrained decoding, retrieval-augmented few-shot (+3.7 pp), LoRA pipeline (never run), `recommend(task, device, bar)` |
| `armore-verify` | Independent TypeScript verifier for receipts, egress receipts, release attestations | 47 TS files, 25 commits, idle since 2026-09-10 | A pluggable `Check` registry where every check must prove its own red — the natural home for model attestations |

### 2.1 The AI subsystem today

| Surface | How it works today [code] | What matters for learning |
|---|---|---|
| **Ask (RAG)** | 800/150-char chunks; all-MiniLM-L6-v2 (384-d) embeddings in a flat JSON index; brute-force cosine top-20 → bge-reranker-v2-m3 (or MMR) → k = 4; abstains below cosine 0.25; cited answer + signed receipt | Conversations saved (`conversationStore.ts`); citation clicks not tracked; no rating |
| **Extract** | Templates invoice / contract / generic_form; Tesseract first, Qwen2.5-VL re-read on low confidence; **greedy decoding, no grammar** (the GBNF sampler is disabled because it can abort the process); `groundExtraction` nulls values not found in the document | Output is read-only, so **no corrections are captured**. The receipt binds the BLAKE3 of the weights and of the input — the right pattern |
| **Assistant (default "fleet agent")** | `agent.ts` loop on the code-slot model; 12 read tools + 6 approval-gated tools + MCP; top-5 memory facts injected | Approvals logged in a signed `audit.jsonl`; history is in-memory only |
| **Coder** | Tool loop with repo RAG, PageRank repo map, one file at a time; escalates to a bigger model or a paired device; per-hunk review | `code-history.json` holds instruction, diff, steps and model — the richest raw signal you have. Rejected hunks are not logged |
| **Deep research** | Sub-question split → consented web meta-search + RAG → low-temperature synthesis | Reports saved locally |
| **Memory** | Only facts the user typed, plus the coder's `.armore/memory.md` | No learned memory |
| **Federated** | Retrieval across paired devices (embedding query out; bounded snippet text back); **federated evaluation already exists** (`federatedEval.ts`, FL-1 #443: scores only) | The in-mesh evaluation half of L2 is partly built |
| **Models** | Hard-coded 24-entry catalog with licence classes; static RAM-tier `recommend()`; sha256 re-checked before every load; **no signed model manifest — changing a recommended model needs an app release** | No channel exists yet to ship improved models |
| **Routing** | `computeRouter.ts` ladder (proceed → download → peer → pooled → TEE → refuse); the TEE rung cannot fire; the per-model `expectedScore` slot is never filled | **Nothing is learned**; `route-jobs.json` is written but never read |
| **Evals** | Two disjoint extraction harnesses — `edge-model-engine` (30 invoices + 12 receipts, Ollama + GBNF) and `crates/drive/src/eval.rs` (33 contracts + 7 invoices + 2 forms, product runtime) — plus coder ladders with hidden tests | No results store and no trend tracking; CI proves models *load*, not how *well* they do |
| **Telemetry** | None leaves the device (crashpad `uploadToServer: false`; no analytics SDK) | Consistent with your rules — and with this design |

**Five facts that shape the design:**

1. Capture is the missing foundation (§5.1).
2. The runtime can already load LoRA adapters, so the inference half of L1 is wiring, not research (§5.4).
3. In-mesh federated evaluation already exists (§5.5).
4. The engine's headline numbers come from a decoding mode the product does not ship (§7, F4).
5. No channel exists to ship a better model without an app release (§5.6).

---

## 3. The copy threat model

### 3.1 What a copier gets for free

- **The code.** The core and SDK once the source flip happens. The desktop app's JavaScript today: it ships inside every public download (`armore-edge-public`, v1.0.10).
- **The primitives.** Iroh, Automerge, llama.cpp, Tesseract and every open-weight model you ship are public.
- **The positioning and UX.** Visible to anyone who installs the app or reads a published doc.
- **The architecture.** Once `docs/papers/` and the repo go public.

[estimate] A funded team starting from a fork could ship a credible clone of today's product in **one to three months**. Any defence that relies on them *not* doing this is already lost; plan for it.

### 3.2 Who would copy, and by what route

Your competitive-landscape chapter (`docs/book/part2/07-the-competitive-landscape.md`) already maps this. I agree with its central call — "commoditisation is more likely than competition" — and add what each actor *cannot* get from a fork.

| Actor | Copy route (from your docs) | What the learning envelope denies them |
|---|---|---|
| **Proton** — your "closest analogue" | "If they ship it, they win it instantly": brand, paying base, privacy credibility | A labelled corpus for your document classes, verifier packs, a factory, a signed eval history |
| **Local-AI apps** — Ollama, LM Studio, Jan, GPT4All, Obsidian's ecosystem, Khoj (closest prior art), Off Grid AI | Any tier-1 gap is closable "in a quarter"; a DIY Tailscale + Ollama gets "roughly that", minus provenance | Per-workspace learning; attested models; task-specific quality that improves monthly |
| **Cloud AI** — OpenAI, Claude for Work, M365 Copilot, Google, Bedrock-in-VPC | "The real competition for revenue"; Bedrock-in-VPC is "the sovereignty argument itself, for most buyers" | Nothing in the open market — but they are **barred** from legally air-gapped deployments, which is where your learning compounds without them |
| **Platforms** — Apple, Google, Microsoft on-device AI | Already ship on-device inference; "no commercial reason" to ship provenance — "disinterest is not a moat… it is a head start" | Cross-vendor, user-owned learning across Mac, Windows, Linux, iOS and Android, with auditor-grade proofs |
| **Vertical incumbents** — PreVeil (no AI, ~$30–32/seat), Clio, Harvey / CoCounsel | Add on-device AI to an existing customer base | Your verifier and eval packs for their document types, built before they start |
| **Standards** — W3C VC, SCITT, C2PA, EU AI Act | Make receipts mechanisable by anyone (`docs/gtm/competitor-analysis.md`) | Receipt *formats* become commodity; years of signed eval and release *history* do not |

### 3.3 Your own moat candidates — and why each needs a loop

Your docs name six different moats. Most are **static** — true on the day they ship, then eroding — and your own competitor analysis calls the strongest of them, mechanised privacy, "a wasting asset". The learning loop is what turns each one into something that **compounds**:

| Moat your docs claim | Evidence status (`docs/strategy/usp-witness-ledger-2026-10-16.md`, `docs/strategy/ROADMAP.md`) | Static or compounding today | What the loop adds |
|---|---|---|---|
| **Legal exclusion** — "the moat is legal, not technical" | Structurally real; narrow market | Static | The barred market is where your corpus and adapters grow without cloud competition |
| **Provenance / receipts** | Partly witnessed (47/47 signatures on v1.0.10; the self-check needs a tamper control) | Wasting, as standards arrive | Turns formats into *history*: signed eval and release attestations accumulate (§6.3) |
| **The mesh** — "THE differentiator" | "Currently unevidenced": X1 *measured as not occurring* | Not yet real | Gives the mesh a measurable job — the private training and evaluation substrate (§5.4–§5.5) |
| **Confidential compute** — "that is the moat" | No TEE ships; grade 2 is unwitnessed | Not yet real | Not needed: the loop works without trusting any server (local DP, in-mesh aggregation) |
| **Federated learning (IDF-1..5)** | Patent-pending; filing status unconfirmed (§7, F0) | Depends on filings | The loop supplies FL's missing inputs: training data (learning events), an eval gate, attestation |
| **Honesty culture** | Real and visible | Compounds slowly | Becomes machine-checkable: every quality claim ships with an attested eval |

### 3.4 What they cannot copy: the slope

```
Quality_Armore(t) = Q0 + ∫ v_Armore(τ) dτ
Quality_fork(t)   = Q0 + ∫ v_fork(τ)   dτ        Q0 = whatever was forked

learning velocity   v = S × Y × C × E
  S  verified learning signals per week   (user corrections, verifier-passed outputs,
                                            synthetic documents, partner-donated documents)
  Y  yield: the share that survives quality filters into train / eval sets
  C  train → evaluate → ship cycles per month
  E  average quality gain per cycle on the hidden north-star evals
```

A fork starts with **S ≈ 0** (no users, no opted-in fleet), **Y ≈ 0** (no domain verifiers, no labelled corpus), **C slow** (no factory) and **E unmeasurable** (no hidden eval). Every term is an asset you build and they cannot fork. The moat is the area between the two curves. The strategic job is to make `v_Armore` large and keep its four inputs closed.

The same equation shows why "hide the code" fails as a strategy: code only sets `Q0`, and `Q0` is exactly what a fork receives.

---

## 4. Design principles: learning inside a sovereignty product

Most data-flywheel playbooks assume you collect user data centrally. Armore cannot — "no cloud sees your data" *is* the product, and ADR-0009 puts external AI APIs "permanently off the table" [doc, quoted in `bench/distill/README.md`]. These seven principles make learning compatible with that promise.

1. **Move the learning to the data, never the data to the learning.**
2. **Every learning signal is the user's data.** It lives in their workspace, encrypted like any document, shows up in `armore privacy list`, leaves with `export`, dies with `purge` — and deleting a source document propagates to everything derived from it (exemplars dropped, adapters retrained or discarded).
3. **Only proofs and noised aggregates ever leave — opt-in, with a `consentId`, and with a row in the egress ledger** (`payload = observedLeaving`). The default is off, which respects your "no telemetry by default" rule (`docs/strategy/path-to-100.md`). Each new outbound report is registered as a `consented` site in the machine-checked `docs/egress-inventory.json` (71 sites today, CI-checked by `ci-develop.yml`) *before* it ships. This reuses contracts you already wrote: desktop already requires a `consentId` whenever content leaves (`armore-edge-mobile/docs/design/035-egress-ledger-axis-contract.md` §6). **L0, L1 and L3 need none of this — only L2's cross-user tier does, and it needs an explicit operator ruling first.**
4. **No improvement without a measurement.** A model or adapter activates only after beating the incumbent on held-out data — globally in the factory, locally on the home node.
5. **Every model is signed and attested, every inference receipt names its model, and rollback is one slot switch.** `provision.rs` already keeps the previous verified model on disk during an update and separates "present" from "active".
6. **One implementation.** Capture, verifiers and exemplar retrieval live in the shared Rust core and reach every surface through NAPI and UniFFI. In your own words, "three implementations of a fail-closed egress ceiling is three chances to fail open" — the same holds for learning.
7. **Honest claims about learning.** "Armore improved from your corrections" must be measured on held-out data, just as the egress ledger refuses to say `observedNotLeaving` without a witness.

---

## 5. The architecture: four learning loops

```
 ┌──────────────────────── L3  MODEL FACTORY  (Armore infra · zero customer data) ────────────────────────┐
 │  new open base models ─► licence-lineage gate ─► GGUF + imatrix quant ─► sweep per device tier          │
 │  synthetic documents ─► open-weight teacher ─► SFT / LoRA / distil ─► eval gates ─► signed attestation   │
 └──────────────▲──────────────────────────────────────────────────────────────────────────┬──────────────┘
                │ federated-eval verdicts, DP failure taxonomy, router priors               │ signed base models,
                │ (opt-in · consent-receipted · aggregates only)                            │ adapters, verifier packs,
 ┌──────────────┴──────────────── L2  MESH EVAL+FLEET ─────────────────────────┐            │ router priors
 │  federated evaluation · federated analytics · later: FL adapters + secure agg│            │ (TUF / armore-release)
 └──────────────▲──────────────────────────────────────────────────────────────┘            ▼
 ┌──────────────┴──────────────── L1  WORKSPACE / HOME NODE  (Pro) ───────────────────────────────────────┐
 │  idle-time LoRA on the workspace's verified examples ─► local eval gate ─► adapter ─► mesh distribution  │
 └──────────────▲──────────────────────────────────────────────────────────────────────────┬──────────────┘
                │ signed learning events (CRDT, encrypted, never leave the mesh)            │ workspace adapter
 ┌──────────────┴──────────────── L0  DEVICE  (Free) ───────────────────────────────────────▼──────────────┐
 │  capture corrections / accepts / regenerations · verifiers · exemplar memory · best-of-N · router bandit │
 └────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 5.1 The foundation: a learning-event layer

Nothing downstream works without capture. Every AI surface emits a **learning event** whenever the user reacts to an output. Strong signals become gold labels; weak signals become measurements.

| Surface | Strong signal (label) | Weak signal | Verifier available |
|---|---|---|---|
| **Extract** | user edits a field (gold); accepts unchanged (silver) | regenerate, export, copy | yes — arithmetic, checksums, grounding (Appendix C) |
| **Vision / OCR** | user fixes recognised text | re-scan | VLM vs Tesseract agreement |
| **Ask** | thumbs up/down; user fixes the answer | follow-up rephrase (negative), citation opened (positive) | citation grounding, self-consistency |
| **Code agent** | tests pass/fail; user keeps or reverts the diff | edits after the agent finishes | the tests themselves |
| **Transcription** | user edits the transcript | — | decoder confidence |
| **Router** | verified success, latency, thermal, battery | — | the outcome itself |

**Implementation:** a new crate, `crates/learn`, in internal's workspace, holding the event store, verifiers and exemplar index, exported through NAPI (desktop) and UniFFI (mobile). Events are signed Automerge changes in the user's workspace — the same trust boundary as the document they describe. The schema is in Appendix A. Keep `crates/learn` separate from the `train` crate that `docs/STRATEGY.md` W4 reserves for adapter exchange and secure aggregation.

**Start from what is already recorded** [measured]. These can be backfilled as learning events on day one:

- `code-history.json` (instruction, diff, steps, model);
- the signed approvals log (`act/<ws>/audit.jsonl`);
- `route-jobs.json`;
- saved Ask conversations.

Two things need new code:

- **Extraction needs new UI:** inline editing of each extracted field, where every edit becomes a gold event. This is the single highest-value missing piece in the product.
- **Rejected code hunks need one log line** at the point they are reverted (`ipc/agentLoop.ts`).

### 5.2 Verifiers: free labels at inference time

Your own measurements show that a small model plus a strong verifier beats a bigger model on its own: verify-driven methods took code from 47% to 73% (`docs/strategy/edge-accuracy-research.md`) [doc]. Verifiers do three jobs:

1. **Select at inference.** Best-of-N per field, majority vote, mesh fan-out.
2. **Direct the user's attention.** Highlight the fields that failed a verifier ("check this"). That prevents silent errors, and it spends the user's correction effort where it is most informative — active learning for free.
3. **Label.** Verifier-pass + user-accept = **gold**. Verifier-pass, unreviewed = **silver**. Verifier-fail = a negative example for preference training.

Extraction verifiers are unusually strong because invoices, receipts and forms carry internal redundancy: arithmetic, check digits, date logic, and the requirement that every value appears in the source text. The catalog is in Appendix C.

**What already exists** [measured]:

- `groundExtraction` already nulls any value with no matching token in the document. That is Appendix C's grounding verifier, in production.
- Tool calls already use validate-and-retry.
- Best-of-N and ensembling scaffolding exists but has **no production importer**.
- The Rust abstention gate is only constructed in `extract_pipeline.rs`, which has no production caller.

Shipped extraction also runs **without** constrained decoding: the GBNF sampler on the pinned `llama-cpp-2` 0.1.150 can abort the process, so it is disabled. Verifiers therefore carry more weight in the product than the engine's grammar-on benchmarks suggest.

**Verifier packs are themselves a closed asset.** A per-locale, per-vertical pack (GSTIN check character, UK VAT check digits, export-control marking detectors, …) encodes domain knowledge a fork has to rediscover one bug report at a time.

### 5.3 L0 — the device loop (Free · no training · zero egress)

1. **Exemplar memory.** Port `edge-model-engine/engine/rag.py`'s `CorpusRetriever.retrieve_external` into `crates/learn`. Index user-verified (document, extraction) pairs with the embedder you already ship (all-MiniLM-L6-v2, the default retrieval encoder), retrieve the k nearest at extraction time, and inject them as few-shot turns.
   - *Evidence:* with generic exemplars the engine measured **+3.7 pp** on `gemma2:2b` (94.1% → 97.8%) [measured, `benchmarks/rag_accuracy_delta.md`]. With the user's own vendors and layouts the lift should be larger [estimate — measure it].
   - *Cost:* prompt length. The same run took 32.6 s → 61.4 s per document on CPU. Use adaptive k (k = 0 when the first sample passes every verifier) and cache the exemplars' KV state.
2. **Entity memory.** Per-vendor facts learned from verified documents (tax ID, address, usual currency and tax rate, invoice-number pattern), used as verifiers and as pre-fill.
3. **Best-of-N with verifiers.** Draw N = 3–5 temperature-diverse samples, but only when the first sample fails a verifier. Fan out to idle mesh peers when they are present. The scaffolding exists; it needs a production caller.
4. **Router calibration.** A per-device contextual bandit chooses between {local model A, local model B, route to home node}. Reward = verified success − λ·latency − μ·battery/thermal cost. It starts from shipped priors (L2) and adapts locally. The inputs already exist [measured]: `route-jobs.json` logs tier, latency and tokens for every job, and `computeRouter.ts` has a per-model `expectedScore` slot that nothing fills. Mobile design 022's deterministic thresholds make a sensible prior; the bandit only learns corrections to them. ⚠ Your novelty guard protects capability-based selection, so this may be IDF-4 territory. Clear it with counsel before it goes into open code.
5. **Measured, visible learning.** Hold out every fifth user-verified document as a private eval and show *"Accuracy on your invoices: 91% → 96% after 140 corrections"* — computed, never claimed.

**What L0 buys:** personalisation from day one for Free users, at zero egress, with no GPU. It is also the capture foundation every other loop depends on.

### 5.4 L1 — the workspace loop on the home node (Pro)

| Step | Design |
|---|---|
| **Trigger** | ≥ 50 new gold examples since the last adapter **and** the home node is idle, on AC power, below a thermal threshold |
| **Data recipe** | gold (user-verified) + silver (passes every verifier and agrees across N samples, capped at ~30%) + a replay buffer of factory gold data so general skills are not forgotten |
| **Train** | LoRA/QLoRA on the current base — MLX on Apple Silicon (most of your fleet), unsloth/PEFT on CUDA. [estimate] A few hundred examples on a 3–8B 4-bit base is minutes to under an hour on an M-series Mac. |
| **Gate** | Beat the active adapter on the workspace's held-out **gold** set by ≥ 0.5 pp, with no field regressing more than 1 pp and no latency regression; otherwise discard |
| **Ship** | Adapter (tens of MB) signed by the home node's device key → moved by drive's encrypted chunked transfer (once the mesh is witnessed; until then it serves the machine that trained it) → activated through `provision.rs`-style slots (the previous adapter stays for instant rollback) → llama.cpp applies the LoRA at load time, so every device keeps one base model |
| **Prove** | `modelServingHash` must cover the base + adapter *digests* (today it digests only the catalog id — §6.3), so the evidence bundle names the exact adapter bytes behind each output |

**This is your FL-2, with its missing inputs.** `docs/PLAN-COMPREHENSIVE.md` (Phase C) already plans FL-2, "adapter exchange": "train a LoRA adapter locally, sign it, offer it to workspace members; peers verify + hot-load". The code is both further along and less finished than the plan suggests [measured]:

- **The inference half exists.** `set_lora` / `apply_lora` in `crates/drive/src/extraction.rs` call llama-cpp-2's `lora_adapter_init` / `lora_adapter_set` on every generation context, and the SDK exposes `LocalLlm.setLora` / `clearLora`. But nothing in the app calls them.
- `adapterExchange.ts` imports, sha-checks and lists adapter manifests, yet records every signature as `'unverified'`.
- There is no trainer in the repo.

FL-2 therefore lacks four things, and L1 supplies them:

- training data — the learning events of §5.1;
- a trainer that has actually run — §7, F2;
- an eval gate — plus in-mesh federated evaluation, §5.5;
- real signature verification on import.

**Don't make L1 wait for the mesh.** Your USP ledger records the cross-device mesh as "measured as not occurring" today. L1 v1 trains on the user's own main machine overnight and serves that machine. Mesh distribution is added once the mesh is witnessed.

**Self-training is the "autoregressive" part.** The system's own *verified* outputs on unlabelled workspace documents become the next round's training data (rejection-sampling fine-tuning, STaR-style). Guardrails against model collapse:

- silver data never enters the eval split;
- the silver share is capped;
- training stops when the held-out gold metric stalls or drops;
- every factory release resets the adapter to the new base and retrains it, so adapters cannot drift for ever.

**Privacy.** An adapter trained on a workspace can memorise that workspace's contents. Treat it as workspace data: same encryption, same membership boundary, included in privacy export/purge, retrained when its source documents are purged. It never goes to Armore.

**Why this is the Pro story.** It turns the Home AI node (#186) from "your idle machine serves the household" into **"your idle machine teaches your AI your documents — and nothing leaves your house."** It also *earns* retention rather than locking it in, which is what your pricing chapter requires (`docs/book/part2/08-capitalizing-on-this.md` §6A.2): the user's data and corrections stay exportable, and what they would give up by leaving is the machinery that keeps compounding them. Phones do not train; they contribute events and consume the workspace adapter.

### 5.5 L2 — federated evaluation, then opt-in fleet signals

Your FL ladder (`docs/PLAN-COMPREHENSIVE.md`, Phase C) already puts **FL-1, federated evaluation,** first, at about a week of effort on the existing job engine. L2 starts there and crosses the user's boundary only later, and only with consent.

**L2a — In-mesh federated evaluation (your FL-1). Nothing leaves the user's devices.** This is already partly built: `federatedEval.ts` (#443) fans the model-eval stage out to every paired device, keeps errored peers in the report rather than dropping them, and moves **only scores** across the wire [measured]. What remains is to point it at learning-event data and use it as the activation gate. Run each candidate (a new factory release, or a new workspace adapter) against the verified examples on *every* device in the workspace via the job engine. Aggregate the scores on the asking device and activate the winner workspace-wide. Egress is the same as today: none beyond the mesh. This is the first thing to build after L0/L1 because it is cheap, it needs no new egress, and it is the gate that makes L1 safe across devices.

**L2b — Cross-user aggregates (opt-in; needs an operator ruling first).** Listed in order of privacy risk, lowest first:

1. **Federated evaluation verdicts.** Armore ships a candidate model C next to the incumbent I (both signed). An opted-in device scores both, at idle, on a sample of its *own* user-verified events and reports only a clipped, noised accuracy delta per task. The aggregate decides rollouts per device tier and locale, and it catches the worst failure of synthetic-trained models: winning on your eval and losing on real documents. [estimate] A few hundred devices per tier is enough to detect a ~2 pp difference.
2. **Failure taxonomy (federated analytics).** Devices report categorical tuples (task, field, error class, layout-cluster ID from an Armore-published codebook, locale) under local differential privacy (randomised response). Aggregated counts tell the factory where to synthesise data.
3. **Router priors.** Per-device-class aggregates of latency, thermal behaviour and verified success warm-start the L0 bandit on new devices.

**L2c — Federated adapter learning (your FL-3/FL-4).** Secure aggregation + DP over adapter deltas. This is the patent-pending lane (IDF-1..5, in `armore-edge-sovereign`). This design only defines the slot it plugs into. Per your own positioning note, the phrase "federated learning" does not enter public copy before FL-2 ships.

**Mechanics for everything in L2b:**

- explicit opt-in per workspace, set by the admin for Team;
- a `consentId` bound into an egress receipt, plus an egress-ledger row (`payload = observedLeaving`, content class "aggregate metric");
- a `consented` entry in `docs/egress-inventory.json`;
- a minimum cohort size;
- a published privacy budget (ε per quarter);
- an "inspect exactly what will be sent" view.

**No trusted server required.** Your docs rule out leaning on an enclave: no TEE ships, the "attested" grade is unwitnessed (all cloud-assist today is grade 3), ADR-0009 is "buy the option, don't exercise it", and an earlier private design rejected an enclave-based aggregator. So L2b uses **local** differential privacy (noise added on the device) plus minimum cohort sizes, which need no trust in the aggregator. ⚠ FL-1 and the L2b items may fall within the scope of IDF-1..5 ("collective model improvement"). Ask counsel before anything here lands in open code.

### 5.6 L3 — the model factory (Armore infrastructure · zero customer data)

This is `edge-model-engine` grown from "prove it once" into a pipeline that runs continuously.

```
new open model release ─► licence-lineage gate ─► GGUF + imatrix quant ─► sweep: hidden evals × device tiers
                                                                                     │ beats incumbent?
synthetic document engine ─► teacher labelling ─► SFT / LoRA / distil ─► eval gates ◄┘
        ▲  targets from the L2 taxonomy                                      │ pass
        │                                                                    ▼
        └──── federated eval (L2) ◄── staged rollout 1% → 10% → 100% ◄── signed attestation + catalog entry
```

1. **Synthetic document engine.** Generate structured truth first, then render it into many layouts, locales and languages — India GST invoices, UK VAT, GCC bilingual Arabic/English, US — and degrade it realistically: phone photos with perspective, shadows and blur; fax; photocopy. `eval/synth_scans.py` is the seed, and its own report says the current renders do not model real optical distortion. That realism gap is where to invest.
2. **Teachers.** Use large **open-weight** models on rented GPUs to label documents and generate rationales. Never use external APIs on customer data (ADR-0009). Avoid API teachers in general: many API terms forbid training competing models on their outputs. Check each teacher's licence for output-training permission.
3. **Training.** SFT/LoRA per task family; distillation into tier-sized students (phone 1–3B, laptop 3–8B, Jetson 7–13B, per `edge-model-engine/docs/MODELS.md`); preference training (DPO) on verifier-ranked pairs.
4. **Continuous base-model adoption.** A watcher on new open releases runs: licence-lineage check (upstream card plus the full `base_model` chain) → GGUF + imatrix quant → sweep on hidden evals per tier → retrain adapters → attest → canary. **Target time-to-adopt: under 14 days.** Permissive small candidates worth sweeping now to replace `qwen2.5:3b` at the bottom of the laptop/phone tiers: **Qwen3-4B (Apache-2.0)** and **Phi-4-mini-instruct (MIT)** [external: upstream cards checked 2026-10-29]. Their accuracy on your task is unmeasured.
5. **Hidden eval corpus.** Versioned, stratified by task, locale, layout and difficulty, and contamination-guarded (canary strings, n-gram overlap checks against training data). Publish a ~10% public subset so headline claims stay reproducible by anyone.
6. **Device farm.** Real hardware per tier (an M-series Mac, a Snapdragon 8-class Android phone, an iPhone Pro, a Jetson Orin Nano) running nightly latency, memory and thermal benchmarks. This replaces latency numbers taken on a shared, contended machine (`benchmarks/ocr_scan_eval.md` had to discard its latency for exactly that reason).
7. **Release.** A Sigstore/in-toto model attestation (Appendix B), verified by a new `armore-verify` check *and* by provisioning before activation; TUF / `armore-release` anti-rollback; staged rollout with federated evaluation as the canary signal. The factory emits the product's **catalog entries** directly, which ends the hand-copied `extractionAccuracyPct`. **This channel does not exist yet** [measured]: there is no signed model manifest, `crates/release` signs app releases only, and a new recommended model needs an app release (`modelManager.ts`, `modelRegistry.ts`). Build a signed, TUF-style **model feed** so models ship on their own cadence. Without it, the release cadence in §8 is capped at your app release cadence.

**Cost [estimate]:** a QLoRA run of a 3–8B model on a few thousand examples is roughly 1–2 GPU-hours on a single A100/H100-class GPU — single-digit dollars per run at typical on-demand rates. Synthetic generation with a 30B-class MoE teacher produces thousands of documents per GPU-hour. A serious monthly release cadence fits in a low-four-figure monthly budget. This also permanently unblocks issue #12, which has waited since July for "one founder Colab run".

### 5.7 L4 — enterprise and design partners

- **On-prem training pack** (a proprietary Compliance pack): the L1 machinery for an organisation's own node or cluster, with Armore's recipes and the organisation's private eval split. Data never leaves the organisation.
- **Recipe telemetry, by contract.** Which recipe variants won locally — hyperparameters, data-mix ratios, verifier thresholds; never data — flows back to improve the factory. This is learning *how to learn* across customers whose data you never see.
- **Data-donation programme.** Partners may contribute documents under contract, redacted on-device first, to the hidden eval corpus in exchange for pricing or early access. Every donated document carries its contract ID in the data manifest.

### 5.8 The flywheel, end to end

> The opted-in fleet's taxonomy shows that *line items on bilingual Arabic/English invoices fail 23% of the time on the phone tier* → the factory synthesises 5,000 such documents, trains, and federated evaluation confirms +9 pp on real devices → a signed release ships → every workspace adapter now starts from a better base → users correct less → they trust Armore with more document types → more signals and more opt-ins → next turn.
>
> *(Illustrative numbers.)* A fork sees none of these arrows.

---

## 6. Defensibility engineering: making copies worthless

### 6.1 Draw the open/closed line for learned assets — now

**Where things stand.**

- No source is public. The #783 source-flip gate waits on provisional filings and counsel sign-off.
- What *is* public: the release repo `armore-edge-public` (v1.0.10 binaries plus `verify-release.sh`) and `armore-claims-canon`.
- The OSS export manifest would ship 896 of 2,237 files, but it is unratified and the git history is unscrubbed (`docs/strategy/oss-export-manifest-2026-09-12.md`).
- ADR-0023 (publication boundary) is still Proposed, and it escalates anything touching `crates/`, model code or FL to the operator.

**That is an opportunity.** The line for learned assets can be drawn *before* the first public commit instead of being clawed back after it. Add the closed paths below to the manifest's forbidden list now, and keep those assets in a separate private repository.

| Asset | Recommended status | Why |
|---|---|---|
| Rust core, SDK, mesh, crypto, receipts | **Open** (licence per F0; ADR-0027 keeps `armore_net` / `armore_link` open) | The trust claim must be auditable |
| `armore-verify`, wire formats, attestation spec | **Open** — publish | Independent verification; become the standard others are measured against |
| Eval **harness** code | **Open** | Makes your claims reproducible, and so believable |
| Public eval subset (~10%) | **Open** | Lets anyone reproduce headline numbers |
| Hidden eval corpus | **Closed** (trade secret) | Prevents copying and Goodhart; a fork ships blind without it |
| Synthetic engine configs, templates, seeds; failure taxonomy | **Closed** | The factory's fuel |
| Training recipes (hyperparameters, curricula, data mixes, verifier thresholds) | **Closed** | Learned know-how |
| Current-generation adapters and fine-tuned weights | **Closed**, licence-bound delivery (Pro/Team) | The product of the loop |
| Previous-generation adapters (N−1) | **Open weights after a lag** | Adoption and goodwill; a stale snapshot costs you little |
| Router priors and policies | **Closed** (signed policy files) | Learned from the fleet |
| Vertical packs (templates, verifier packs, eval splits, compliance mappings) | **Proprietary** (already planned) | The B2B moat |
| FL mechanisms | **Closed until filed** (already in `-sovereign`) | Absolute novelty |

This is consistent with `docs/STRATEGY.md` §7. Features stay ungated, and Pro already lists **"hosted model delivery"**: continuously improved models *are* hosted model delivery. Free keeps the open base models, L0 learning and N−1 adapters. Pro adds the monthly Armore-tuned models and home-node learning. Team adds organisation adapters, in-org federated evaluation and attested evidence. The verifier and file formats stay free forever, as your pricing chapter requires.

**It also closes a monetisation gap your docs record.**

- Today Pro differs from Free by one boolean (`alwaysOnNode`), and Team and Enterprise are byte-identical to Pro (`docs/strategy/tier-differential-2026-08-30.md`).
- The licence issuing key is empty, so every install resolves to Free, and only `alwaysOnNode` can be refused server-side (`docs/licence-issuance-runbook.md`).
- In an open client, a client-side gate runs on the honour system.

**Hosted model delivery is enforceable for the same reason the durability depot is: the asset lives on infrastructure you run.** Monthly attested models (Pro) and organisation adapters plus in-org evaluation (Team) give each paid tier a real, server-side difference, without gating a single client feature. They also respect "no free feature may cost server money" (`docs/v2/DECISIONS.md`): L0 costs nothing, and N−1 weights can sit on a free public model hub.

### 6.2 Protecting shipped weights: what works, honestly

On a general-purpose device, a determined attacker can extract anything the device can run. Raise the cost, make leaks traceable, and rely on freshness:

1. **Licence-bound delivery.** Current-generation adapters download only with a valid Pro/Team licence, as signed manifests through the same TUF / `armore-release` path as app updates.
2. **Encryption at rest under the device keystore** (Secure Enclave, StrongBox, DPAPI, OS keychain), decrypted only into process memory. Note that llama.cpp normally memory-maps files, so this needs an in-memory load path.
3. **Per-licence fingerprinting.** Unique low-magnitude weight perturbations, or trigger-response watermarks, make a leaked adapter traceable to the licence it came from — legal leverage, not prevention.
4. **An explicit weights licence, separate from the code licence:** no redistribution, no use to train competing models.
5. **Shelf life — the defence that actually works.** With monthly releases, a stolen adapter is stale within weeks.

Do **not** spend effort obfuscating the open core or the app's JavaScript (it ships as a plain Electron asar today, with no obfuscation step [measured: `packages/app/electron-builder.yml`]). A trust product is meant to be read. Keep what must stay closed *off the device* instead (§6.1).

### 6.3 Proof-carrying models: turn your verification culture into the moat

Every model release ships with a signed attestation (Appendix B) that binds together:

- model and adapter digests;
- the upstream base model and its licence;
- the training-data manifest by class (synthetic / public / partner-donated; `customer_data: false`);
- eval results per suite and device tier, with the harness commit;
- the gate thresholds;
- the previous release.

The pattern already exists in one place and is missing in another [measured]:

- **Extraction receipts get it right.** They bind the BLAKE3 of the model weights *and* of the input.
- **The per-inference egress receipt does not.** Its `modelServingHash` is `servingHash(outcome.decision.target?.id ?? "local-model")` (`packages/app/src/main.ts:1876`; `servingHash` is a canonical-JSON sha256, `egressReceipt.ts:146`). For an on-device run the router's target is `null`, so **every on-device receipt hashes the constant string `"local-model"`**: it identifies neither the model nor its bytes.

Generalise the extraction pattern: the serving hash should cover the attested weight digests (base + adapter + quant + runtime version). The catalog already records sha256 for pinned entries, and `assertModelUnchanged` re-hashes before every load. Then resolve the hash to the attestation, and the evidence bundle can say: *"this extraction came from adapter v14, which scored 97.3% (n = 1,840, ±0.8) on the invoice suite at laptop tier."* No local-AI competitor can produce that sentence, and a fork cannot backfill 18 months of signed history.

Implementation: a `model-release-attestation` check in `armore-verify`, written to Charter §4 (it ships with the mutations that make it fail), plus the same verification inside provisioning before `activate_model`. Keep the verifier independent of the producer — `armore-verify` exists because internal gates once reported green while failing (Charter §3) — and give model attestations the same treatment from day one. Build it **on** the standards your competitor analysis expects to commoditise receipts — in-toto and Sigstore today, SCITT-style transparency logs and C2PA where they fit — not against them. Once receipts are mechanisable by anyone, the difference is the history you have already signed. When the format is stable, publish it and keep the verifier free forever, as your pricing chapter already rules.

Two guardrails from your own rulings:

- **Keep the engine's cloud baseline internal.** It is a useful engineering signal, but your long-game chapter rules out ever publishing a frontier comparison (§4.3).
- **The proof moat is only as strong as the receipt spine under it.** The USP ledger records an empty `receipts_root` and 5 of 45 egress sites logged. Completing that spine is a prerequisite, not a parallel track.

⚠ **This may already be yours on paper.** Your novelty guard protects categories this section touches — attestations about training runs, commitments to training data, binding outputs to models. That suggests proof-carrying models may already sit inside IDF-1..5. Treat §6.3 as input to that lane, not a parallel build: route it through `armore-edge-sovereign` and counsel before any of it lands in open code.

This document uses none of the guard's blocked tokens. I checked it against the guard's own 56 patterns, using the guard file as a positive control (31 lines matched there, 0 here). Even so, keep it out of the repos.

### 6.4 IP: patents, trade secrets, trademarks, licence hygiene

**What your docs record** (`docs/book/part2/04-the-long-game.md` §4.5, `docs/strategy/ROADMAP.md` R8, `.github/workflows/novel-signature-guard.yml`):

- Five patent-pending FL mechanisms (IDF-1..5) live in the private `armore-edge-sovereign` repo.
- IDF-3 must **not** be filed: a freedom-to-operate conflict with an existing third-party US patent.
- IDF-4 has a §112 enablement gap: the element that carries its eligibility and its only surviving novelty is not yet built.
- The absolute drop-dead for filing was **2026-10-28**. The last in-repo status is "not yet filed" (2026-10-07).
- R8: "our own public releases are prior art against us". Some novelty is recorded as likely spent as of 2026-10-13, which leaves only the US one-year grace period.
- The novelty guard scans about 48 text signatures across every tracked file (docs included), on both PR head and merge result, and fails closed. It reads **text only**: nothing gates *behaviour* that ships compiled in a release.

**What that implies for this strategy** (I am not a lawyer; this is for counsel):

1. **Patents are a supporting moat here, not the primary one.** Even granted, FL patents protect specific mechanisms; they do not stop a competitor learning by other means. The learning envelope compounds with or without them.
2. **Add a release-time patent gate.** Any feature on a counsel-maintained "novel mechanisms" list needs a filing receipt or a written waiver before it ships. The text guard cannot see a mechanism that ships compiled.
3. **Decide the licence with counsel before the source flip.** Apache-2.0 §3 grants every user a licence to contributors' patents that read on their contributions, and terminates it for anyone who sues. MIT has no express patent grant. Your plan to patent FL mechanisms while opening the core makes this choice load-bearing — and today the root `LICENSE` (MIT) contradicts `Cargo.toml` and the SDK (Apache-2.0).

Additional points for the learning layer:

- **Candidate new subject matter from this design** — for counsel to assess; I am not judging novelty:
  - consent-receipted federated evaluation tied to sovereignty grades;
  - proof-carrying adapters that bind inference receipts to eval attestations;
  - verifier-gated self-training on a user-owned device mesh, with deletion propagation;
  - learned capability-aware routing (may overlap IDF-4).

  At least three of these four sit near categories your novelty guard already protects. That is a good sign — you identified this territory before I did — and a reason to coordinate with the `-sovereign` lane rather than file or build in parallel.
- **Decide open or closed per mechanism.** If the core ships under Apache-2.0, contributing an implementation licenses to every user any of your patents that read on it. So either open a mechanism deliberately (as prior-art substrate) or keep it in a closed repo until it is filed. Extend `novel-signature-guard.yml` to cover the learning mechanisms above.
- **Trade secrets.** Keep the §6.1 closed assets in a separate private repository with need-to-know access, employee and contractor agreements, and access logging. List their paths as forbidden in the OSS export pipeline.
- **Trademarks.** Register "Armore" and "Armore Edge" in the target markets (India, US, UK, GCC, EU). A fork can copy code; it cannot copy the name.
- **Model-licence hygiene is itself a moat.** You can sell into regulated buyers because your licence chain is clean and attested. A fork that ships a non-commercial base cannot.

### 6.5 Real network effects — not invented ones

Your long-game chapter rules out tokens, chains and "invented" network effects. The three below are real, because each is a measurable consequence of use rather than an incentive scheme.

- **Workspace.** Every member's correction improves everyone's adapter, so a team gets more value the more it uses Armore. Retention is earned by the accumulated tuning, while the data itself stays exportable.
- **Fleet — only if L2b is ruled in.** Opted-in devices make every release better for everyone. Expect it to be modest at first: its value is release safety and targeting, not a data hoard. A fork without a fleet ships blind, and a competitor that centralises data cannot sell into the sovereign market.
- **Ecosystem.** Vertical packs (templates, verifier packs, eval splits, compliance mappings) as the unit partners build, and the attestation spec and `armore-verify` as the standard everyone conforms to.

### 6.6 Standards and compliance position

Get design partners' auditors to accept evidence bundles. Map attestations to the documentation duties of NIST AI RMF and the EU AI Act — your control-inheritance posture (`docs/STRATEGY.md` §6) already points this way. Each accepted audit is a reference a copier does not have and cannot buy quickly.

### 6.7 Speed as a moat: point the engineering fleet at the loop

Your engineering throughput is extraordinary — about 1,000 commits a month, largely agent-driven. Since September, though, almost none of it has gone into the part that compounds (§1.2). The verification culture is a real asset for the *proof* moat (§6.3), but the *learning* moat needs protected capacity:

- a standing learning lane with its own KPIs (§8);
- the factory running nightly;
- agents triaging failure clusters and proposing data or recipe fixes;
- a human approving each release;
- learning-velocity metrics added to the daily activity summary that CLAUDE.md already asks agents to produce.

---

## 7. Fix-first findings

These block the flywheel. They are ordered by urgency.

**F0 — The IP clock (time-critical, external deadline).**

- Your long-game chapter set **2026-10-28** as the absolute drop-dead for filing. The last in-repo record says "not yet filed" (2026-10-07). Filings are held off-repo by design, so confirm the status with counsel **today**.
- If nothing was filed, assume India/EU novelty is gone for anything already embodied in a public release. The US one-year grace period is then the only remaining net (`docs/strategy/ROADMAP.md` R8).
- In the same conversation, settle the core licence (MIT in `LICENSE` vs Apache-2.0 in `Cargo.toml` and the SDK, undecided per ADR-0027) **before** the source flip. Add the release-time patent gate from §6.4.

**F1 — Licence lineage (blocking; verified 2026-10-29).**

| What | Evidence |
|---|---|
| Default VLM is Qwen2.5-VL-3B | `packages/app/src/defaultModel.ts:374` (`RECOMMENDED_VISION` = `default-vlm-qwen25vl3b`); Rust defaults in `crates/drive/src/ocr.rs:1262`, `vlm.rs:281`, `extraction.rs:1844` [measured] |
| Upstream licence is non-commercial | Hugging Face API, `Qwen/Qwen2.5-VL-3B-Instruct`: `license_name: qwen-research` [external] |
| Catalog says Apache-2.0 | `modelCatalog.ts` entry `qwen2.5-vl-3b-instruct-q4` cites "HF card license apache-2.0", read from `ggml-org/Qwen2.5-VL-3B-Instruct-GGUF`, a re-upload whose own card names `base_model: Qwen/Qwen2.5-VL-3B-Instruct` [measured + external] |
| Engine recommends a blocked model | `edge-model-engine/engine/README.md` example → `qwen2.5:3b`; `engine/finetune` falls back to `qwen2.5-3b`; your app classes `qwen2.5-3b-instruct-q4` as `Qwen-Research` / `non-commercial` and blocks it [measured] |

**Actions:**

- switch the default VLM to an upstream-permissive model that fits the RAM tier (Qwen2.5-VL-7B is Apache-2.0 upstream [external]; for small-RAM devices, sweep permissive small VLMs);
- add a catalog CI check that walks the `base_model` chain to the **upstream** card and fails on a mismatch;
- give the engine a licence filter that imports the app's `licenseClass` table, so `recommend()` only returns shippable configs;
- have counsel confirm the remediation for builds already distributed.

**F2 — The fine-tune has never run.** Merge `fix/12-colab-notebook-preflight`. Replace the manual Colab hand-off with a scripted GPU job (§5.6). Produce adapter v1 on a **permissive** base. Never score it on training documents. Report it on the manifest's held-out split *with its confidence interval*, and treat it as a pipeline proof rather than a quality claim until F3's larger corpus exists.

**F3 — The eval corpus is too small and too synthetic to steer by.**

- The invoice suite scores 30 documents × 8 fields = 240 field instances. At 95.4% that is a 95% confidence interval of roughly **±2.7 pp**, and wider in practice, because errors cluster within documents. So `qwen2.5:3b` "clearing" the 95% bar at 95.4% is inside the noise. The engine's own fine-tune README half-says this ("a slightly harder held-out set could plausibly dip under it").
- The fine-tune's held-out split is 6 invoices — 48 fields, roughly ±6 pp. Issue #12 ("prove the fine-tuned model clears the bar") **cannot be proven on it** at a 95% bar, whatever the model does.
- The scanned sets are synthetic renders, and the OCR report says they miss real optical distortion.

Target: ≥ 2,000 hidden documents within three months, across invoice, receipt, contract and form, multiple locales, and real-photo degradation (§5.6). Report confidence intervals next to every number in `benchmarks/`, and make `recommend()` require the *lower* bound to clear the bar.

**F4 — Engine ↔ product split: the numbers don't describe what ships** [measured].

- The engine measures accuracy through Ollama **with** GBNF constrained decoding. Shipped extraction runs greedy **without** a grammar, because the sampler is disabled on `llama-cpp-2` 0.1.150 (it can abort the process). The engine's own delta table puts grammar at up to +3.1 pp on some models, so **shipped extraction accuracy is currently unmeasured**.
- There are two disjoint harnesses and corpora: `edge-model-engine` (30 invoices + 12 receipts) and `crates/drive/src/eval.rs` (33 contracts + 7 invoices + 2 forms). Neither feeds the other. There is no results store and no trend tracking, and the inference CI proves that models *load*, not how *well* they do.
- The only bridge into the product is a hand-copied number.

**Action:**

- one harness, running on the product's own runtime through the SDK (as `bench/distill` already does), over one versioned corpus;
- results stored with commit, model digest and confidence interval;
- the factory emits catalog entries and attestations from those results;
- unblock the grammar through the `llguidance` route the code already names, or a `llama-cpp-2` bump, then re-measure.

**F5 — Feedback capture does not exist** [measured]. Build, in this order:

1. **Inline field editing in Extract.** Every edit becomes a gold learning event (§5.1, Appendix A).
2. **A log line for rejected code hunks.** Today they are reverted silently (`ipc/agentLoop.ts`).
3. **Persisted assistant history.** It is in-memory only today (`assistantHistory.ts`).
4. **`route-jobs.json` read back into the router's `expectedScore`.**
5. **A one-tap rating on Ask answers**, plus tracking of citation opens.

All five stay local, cost no server money, and need no consent change.

**F6 — Mobile has no on-device inference yet.** `docs/mobile-ai-cost-memo.md` keeps #11/#18 gated and records that the missing piece is an operator ruling on engine and platform order, not more measurement. Until that ruling, phones cannot take part in L0. Rule on it.

**F7 — Capacity allocation.** See §6.7: about 0.1% of recent internal commits touched `bench/`. Name an owner and a weekly learning-velocity review.

**F8 — The beachhead decides the first corpus.**

- Your docs give four different first markets: defense/spacetech, legal, global B2C, and "regulated deferred to year 5". A written resolution is due 2026-11-16 (`docs/book/part2/08-capitalizing-on-this.md` §3.2).
- The learning envelope works for any of them, but its first hidden corpus and verifier pack do not: Indian defence-supplier paperwork, GCC legal contracts and consumer receipts are different distributions.
- Resolve the beachhead first; the factory's first corpus follows it.

### 7.1 Outside the moat question, but worth acting on

- **The default code model may not decode on the pinned runtime.** Two ignored tests in `crates/drive/src/extraction.rs` (`:1667`, `:1690`) state that `llama-cpp-2` 0.1.150 "cannot decode Qwen3-30B-A3B (MoE)". `Cargo.lock` pins 0.1.150, and Qwen3-Coder-30B-A3B is the default code model, which also drives the default assistant's tool loop. `docs/limitations.md` does not mention it [measured]. Verify on a shipped build, and check which runtime produced the scorecard's Qwen3-Coder numbers.
- **A claims gap on federated ask.** README line 36 says "only an embedding crosses the link, never the text". The code returns bounded snippet *text* with each result (`crates/link/src/federated.rs`, `FederatedSnippet.snippet`); only the *query* is embedding-only [measured]. That is fine inside an authorised mesh, but the sentence is wrong. Your own claims discipline would correct it before an auditor does.
- **On-device receipts identify no model** (§6.3). This is worth fixing for its own sake, independent of attestations.

---

## 8. Metrics: the learning-velocity dashboard

| Metric | Definition | Measured where | 12-month target [estimate] |
|---|---|---|---|
| **North star: task accuracy** | field-level exact match on the hidden eval, per task × device tier | factory | invoice: laptop ≥ 97%, phone ≥ 95% |
| **Real-world correction rate** | % of extractions the user edits | L0 locally; fleet aggregate via L2 opt-in | halve every 6 months |
| **Time-to-adopt** | new open base release → attested candidate | factory | < 14 days |
| **Release cadence** | attested model releases per month | factory | monthly → bi-weekly |
| **Failure-to-fix cycle time** | taxonomy cell flagged → shipped improvement | factory + L2 | < 30 days |
| **Verified examples per week** | synthetic + donated + verifier-gold | factory | grows every month |
| **Eval coverage** | hidden documents by task × locale × layout | factory | ≥ 2k by month 3, ≥ 10k by month 12 |
| **Workspace adapter lift** | median held-out gain over base after 100 corrections | L1 locally; L2 aggregate | positive and growing |
| **Fleet participation** | % of Pro/Team workspaces opted into federated eval | L2 | enough per tier for decisive verdicts |
| **Attestation coverage** | % of models, and of inference receipts, resolvable to an attestation | armore-verify | 100% |

---

## 9. Roadmap

Phases 0–1 are sized for what you have today — one core engineer plus the agent fleet — because your kill conditions make a second engineer a dated question (K2: 2027-01-31), not an assumption.

| Phase | Window | Deliverables | Exit criteria |
|---|---|---|---|
| **0 — First turn** | weeks 0–4 | F0 counsel check + core-licence decision; F1 licence fix; F2 scripted fine-tune → adapter v1 + signed eval; `crates/learn` v0 (event capture in desktop Extract, covered by privacy list/export/purge); L0 exemplar memory; open/closed manifest; F8 beachhead → first corpus scope | Filing status known; no non-commercial upstream in any default or recommendation; adapter v1 reproducible with one command; measured local lift on a seeded test workspace |
| **1 — Factory + L0 in product** | months 1–3 | One eval harness on the product runtime (F4); synthetic engine v1 (multi-locale, real-photo degradation); hidden corpus ≥ 2k; base-model watcher + per-tier sweeps; **signed model feed** (models ship without an app release); model attestation + `armore-verify` check + provision-time verification; serving hash over weight digests; factory-generated catalog entries; verifier pack v1 + best-of-N on verifier failure; grammar unblock (`llguidance` or a pin bump); mobile #233 ruling | A monthly attested release train, independent of app releases; shipped extraction accuracy measured with a CI; correction rate measured locally |
| **2 — Home-node learning (Pro)** | months 3–6 | L1 trainer (MLX first, CUDA second); wire `LocalLlm.setLora` + real signature checks in `adapterExchange.ts`; local eval gate; L2a gate on `federatedEval.ts`; mesh distribution once witnessed; rollback; receipts naming the adapter; self-training behind a flag; cross-user federated-eval pilot with design partners (after the operator ruling) | Positive median adapter lift; the ledger proves zero adapter egress |
| **3 — Fleet learning** | months 6–12 | Federated analytics → factory targeting; fleet router priors (after the IDF-4 check); FL adapters from `-sovereign` after filings; enterprise on-prem pack + recipe telemetry + donation agreements | ≥ 2 shipped improvements per quarter traceable to fleet signals |
| **4 — Ecosystem** | 12+ months | Vertical model packs (e.g. India legal, DIB/CUI); adapter exchange (`docs/STRATEGY.md` W4); attestation spec published | Third parties building on packs and the spec |

---

## 10. Risks and mitigations

| Risk | Why it matters | Mitigation |
|---|---|---|
| **Privacy leakage through adapters** | An adapter is compressed training data | Adapters never leave the workspace; L2 shares only DP aggregates; canary-insertion memorisation tests before anything crosses a boundary |
| **Model collapse from self-training** | Quality degrades silently | Gold-only eval; silver cap; reset to the factory base each release; diversity monitoring |
| **Goodhart on the eval corpus** | Numbers rise, users do not benefit | Hidden rotating test sets; federated eval on real documents as the canary; correction rate as the north star |
| **Synthetic-to-real gap** | Your own OCR report shows synthetic scans miss real distortion | Invest in realism; partner-donated real documents for eval; federated eval |
| **Licence contamination** | A non-commercial base makes every derivative unsellable | Lineage gate in the factory and catalog CI; the attestation records the licence |
| **Trust backlash ("they train on my data")** | The sovereignty brand is the product | Default off; plain-language consent; the ledger shows every byte; L0 and L1 need no egress at all |
| **Regulation** (EU AI Act documentation, India DPDP consent) | Fines and procurement blockers | Attestations double as technical documentation; consent receipts double as consent records — confirm with counsel |
| **Patent disclosure** | Forfeits novelty | This document is disclosure-gated; extend the novelty guard to the learning mechanisms |
| **Compute budget** | The factory needs GPUs | On-demand rental; track cost per release; small models are cheap to train |
| **Mobile constraints** | Battery, thermal, app-store rules on downloaded models | Phones do L0 and consume adapters; training stays on the home node |
| **Team capacity** | The learning lane is starved today | Protected lane; KPIs in the daily summary |
| **Novelty already spent** | Releases are prior art; the filing date may have been missed | Counsel today (F0). The learning envelope does not depend on patents |
| **Platform commoditisation** | Apple, Google and Microsoft ship on-device AI for free | Compete where platforms have "no commercial reason" to go: cross-vendor, user-owned learning with auditor-grade proofs |
| **Free-tier cost** (your K4, 2026-11-30) | A free feature that costs server money breaks your own rule | L0 is device-only; N−1 weights sit on a free public hub; factory cost is R&D, not per-user |
| **"Small models close the gap" fails** (your Bet 3) | Your docs call it "most likely false" | Nothing here depends on it: the loop competes on task accuracy on the user's own documents, never against frontier models |

---

## Appendix A — Learning-event schema (sketch)

```json
{
  "schema": "armore.learning-event/v1",
  "event_id": "blake3:…",
  "workspace_id": "ws_…",
  "device": "ed25519:<node public key>",
  "at": "2026-10-29T12:00:00Z",
  "task": "extract.invoice",
  "input_ref": "blake3:<document hash — the document itself stays in drive>",
  "model": { "base": "sha256:…", "adapter": "sha256:… | null", "catalog_id": "…" },
  "receipt_ref": "<execution receipt id in chain>",
  "prediction": { "invoice_number": "INV-4O192", "total": "13321.50" },
  "verifiers": [
    { "name": "arith.subtotal_plus_tax_eq_total", "pass": true },
    { "name": "grounded.invoice_number_in_text", "pass": false }
  ],
  "user_action": "edited",
  "correction": { "invoice_number": { "from": "INV-4O192", "to": "INV-40192" } },
  "label_quality": "gold",
  "consent_snapshot": { "fleet_eval": false, "fleet_analytics": false }
}
```

Stored as a signed Automerge change inside the workspace, encrypted at rest, listed by `armore privacy list`, exported by `export`, erased by `purge`. Purging `input_ref` cascades to the event, to exemplar-index entries and to the adapter's retraining queue.

## Appendix B — Model release attestation (sketch, in-toto Statement v1)

```json
{
  "_type": "https://in-toto.io/Statement/v1",
  "subject": [
    { "name": "armore-extract-base-q4_k_m.gguf", "digest": { "sha256": "…" } },
    { "name": "adapter-invoice-v14.gguf",        "digest": { "sha256": "…" } }
  ],
  "predicateType": "https://armore.ai/attestation/model-release/v1",
  "predicate": {
    "base_model": { "id": "<upstream repo id>", "revision": "<upstream commit>",
                    "license": "apache-2.0", "license_source": "upstream-card" },
    "training": { "recipe": "sha256:…", "data_manifest": "sha256:…",
                  "data_classes": ["synthetic", "public-licensed", "partner-donated"],
                  "customer_data": false },
    "quantization": { "method": "imatrix", "type": "Q4_K_M", "calibration_set": "sha256:…" },
    "eval": [
      { "suite": "extract.invoice@v12", "harness": "<factory commit>",
        "metric": "field_exact_match", "value": 0.973, "n": 1840, "ci95": [0.965, 0.980],
        "device_tier": "laptop", "latency_p50_ms": 2100 }
    ],
    "gates": { "max_field_regression_pp": 1.0, "min_aggregate_gain_pp": 0.5, "passed": true },
    "previous": "sha256:<previous release attestation>"
  }
}
```

Signed with Sigstore (keyless, workflow identity) or the `armore-release` offline key. Verified by `armore-verify` (new check, with adversarial cases) and by provisioning before `activate_model`.

## Appendix C — Extraction verifier catalog

| Verifier | Checks | Catches | Strength |
|---|---|---|---|
| Schema | output matches the task schema (already guaranteed by constrained decoding) | malformed output | hard |
| Arithmetic | Σ line items ≈ subtotal; subtotal + tax ≈ total; qty × unit price ≈ amount | misread digits, dropped lines | hard |
| Tax plausibility | tax ÷ subtotal matches a rate valid for the locale *and date* — keep rate tables as dated data (India restructured its GST slabs in 2025) | wrong field mapping | soft |
| Identifier check digits | GSTIN check character, UK VAT check digits, IBAN mod-97, PAN format | OCR digit confusions (0/O, 1/l) | hard |
| Date logic | dates parse; due ≥ issue; not implausibly far from today | swapped or misread dates | soft |
| Grounding | every extracted string appears (fuzzily) in the OCR/text layer | hallucinated values | hard for strings |
| Entity memory | vendor facts match previously verified documents | vendor confusion | soft |
| Cross-engine agreement | VLM extraction vs OCR + LLM extraction agree per field | table failures (your −28.7 pp line-item finding under OCR) | soft |
| Self-consistency | a field's value agrees across N samples | low-confidence fields | soft |

## Appendix D — Evidence index

| Topic | Where |
|---|---|
| Strategy, tiers, open-core, W1–W4, "not now" | `armore-edge-internal/docs/STRATEGY.md`, `docs/strategy/positioning.md` |
| Model scorecard, accuracy research | `docs/strategy/ai-model-scorecard.md`, `docs/strategy/edge-accuracy-research.md` |
| Distillation bench, ADR-0009 external-API ruling | `bench/distill/README.md` |
| Model catalog, licence guard, defaults | `packages/app/src/modelCatalog.ts`, `licenseGuard.ts`, `defaultModel.ts` |
| Mobile strategy, FL patent hygiene, egress ledger | `armore-edge-mobile/docs/strategy/mobile-app-strategy.md`, `docs/design/035-egress-ledger-axis-contract.md` |
| Model provisioning, extraction export | `armore-edge-mobile/rust/armore-core/src/provision.rs`, `extract.rs` |
| Engine evidence | `edge-model-engine/benchmarks/*.md`, `engine/rag.py`, `engine/recommend.py`, `engine/finetune/README.md` |
| Independent verification | `armore-verify/CHARTER.md`, `src/core/checks/index.ts` |
| Competitors, moat claims, "commoditisation more likely than competition" | `docs/book/part2/07-the-competitive-landscape.md`, `02-the-market.md`, `docs/gtm/competitor-analysis.md` |
| Decouple from model quality; BYOM; IP clock (2026-10-28); IDF-3/IDF-4 | `docs/book/part2/04-the-long-game.md` §4.3, §4.5 |
| Witnessed vs unwitnessed claims; mesh X1 "measured as not occurring"; R8 prior art | `docs/strategy/usp-witness-ledger-2026-10-16.md`, `docs/strategy/ROADMAP.md` |
| FL ladder FL-0…FL-4 (federated evaluation = FL-1; adapter exchange = FL-2) | `docs/PLAN-COMPREHENSIVE.md`, Phase C |
| Tiers, licence issuance and enforcement, pricing | `docs/strategy/tier-differential-2026-08-30.md`, `docs/licence-issuance-runbook.md`, `docs/book/part2/08-capitalizing-on-this.md` |
| Kill conditions K1–K6 | `docs/strategy/KEEP-OR-KILL-CONDITIONS.md` |
| Publication boundary, source flip, export manifest | `docs/adrs/0023-…`, `docs/strategy/oss-export-manifest-2026-09-12.md` |
| TEE / cloud-assist rulings | `docs/adrs/0009-tee-confidential-compute-for-cloud-assist.md` |
| Egress inventory (71 sites, CI-checked) | `docs/egress-inventory.json`, `.github/workflows/ci-develop.yml` |
| Licence file contradiction | `LICENSE` (MIT) vs `Cargo.toml:31` (Apache-2.0) |
| Ask / RAG path and assistant wiring | `packages/app/src/main.ts` (Ask ≈ 14171–14530, assistant ≈ 17560–17690), `rag.ts` |
| Extraction runtime: grammar disabled, LoRA hooks, MoE pin notes | `crates/drive/src/extraction.rs` (≈ 1660–1700, 2540–2560, 2741–2774); `Cargo.lock` (`llama-cpp-2` 0.1.150) |
| Extract UI (read-only output, tamper demo) | `packages/app/src/renderer/react/routes/Extract.tsx` |
| Routing ladder, unused `expectedScore`, on-device `target: null` | `packages/app/src/computeRouter.ts`, `routedInferenceService.ts` |
| Model defaults, catalog, recommendation, load-time re-hash | `packages/app/src/defaultModel.ts`, `modelCatalog.ts`, `modelRecommend.ts`, `jailedChat.ts` (`assertModelUnchanged`) |
| Signals already recorded | `packages/app/src/codeHistory.ts`, `ipc/agentLoop.ts`, `conversationStore.ts`, `assistantHistory.ts` |
| Adapters | `packages/app/src/adapterExchange.ts`, `packages/sdk/src/lib.rs` (`setLora` / `clearLora`) |
| Federated evaluation (FL-1) and federated retrieval | `packages/app/src/federatedEval.ts`, `crates/link/src/federated.rs` |
| Serving hash | `packages/app/src/main.ts:1876`, `egressReceipt.ts:146` |
| Internal extraction eval | `crates/drive/src/eval.rs`, `crates/drive/eval-corpus/` |
| Mobile routing design (static thresholds, not implemented) | `armore-edge-mobile/docs/design/022-*.md` |
| Novelty guard | `.github/workflows/novel-signature-guard.yml` |

## Appendix E — Glossary (one line each)

- **LoRA / adapter:** a small add-on (tens of MB) that specialises a base model without retraining it; swappable per task or workspace.
- **Verifier:** an automatic check of an output (arithmetic, checksums, tests) that needs no human.
- **Best-of-N:** generate several answers and let verifiers pick the best one.
- **Self-training:** the model's own *verified* outputs become training data for its next version.
- **Federated evaluation:** devices test a candidate model on their own data and report only a score.
- **Differential privacy (DP):** adding calibrated noise so no individual's data can be inferred from an aggregate.
- **Secure aggregation:** combining many devices' updates so the server sees only the sum, never an individual update.
- **imatrix quantisation:** compressing a model using a calibration set so it loses less accuracy at the same size.
- **Distillation:** a large "teacher" model generates labels or examples that train a small "student".
- **Attestation:** a signed statement binding a file's digest to claims about it (how it was built, how it scored).
