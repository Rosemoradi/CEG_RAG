# CEG-RAG: Cross-Encoder Reranker Signals for Robust RAG Defense Against Poisoning Attacks

CEG-RAG is a practical defense framework against **corpus/knowledge-base poisoning** in **Retrieval-Augmented Generation (RAG)** systems.

RAG improves factuality by grounding LLM outputs in retrieved evidence—but it also expands the attack surface. If an adversary can inject crafted passages into the knowledge base, those poisoned chunks can be retrieved, ranked highly, and ultimately steer generation toward attacker-chosen targets.

CEG-RAG turns a component many strong RAG pipelines already have into a security sensor: the **cross-encoder reranker**. Instead of treating the reranker as a black box that only outputs relevance scores, CEG-RAG extracts the reranker’s **final-layer pooled [CLS] embeddings** for each query–passage pair and uses them as features for a **multiple-instance learning (MIL)** detector. Retrieved context is modeled as a “bag” and each passage as an “instance,” enabling weak supervision using **context-level labels** while still supporting **chunk-level localization**.

When poisoning is detected, CEG-RAG performs **context repair before generation**: it filters high-risk chunks using learned poison scores and replaces them with safer lower-ranked candidates while maintaining a fixed context budget. This keeps overhead low (no extra LLM calls are required) and prevents malicious evidence from reaching the generator.

## Key ideas

- **Reuse existing signals:** Leverage cross-encoder hidden representations already computed during reranking.
- **MIL for weak supervision:** Train with context/bag labels while still localizing malicious chunks.
- **Pre-generation defense:** Detect + repair the retrieved context before the LLM sees it.
- **Low overhead:** No additional LLM verification calls; defense runs on reranker activations.

## What the pipeline does

1. **Retrieve + rerank** candidate passages for a query using a cross-encoder reranker.
2. **Extract [CLS] embeddings** from the reranker for each query–passage pair.
3. **Run MIL detection/localization** over the retrieved set to:
   - classify whether the context is poisoned (context-level detection)
   - score individual passages (chunk-level localization)
4. **Repair the context** if poisoning is detected by removing high-risk passages and refilling with safer candidates under the same context budget.
5. **Generate** the final answer using the repaired context.

## Results summary (high level)

CEG-RAG is evaluated on **MS-MARCO**, **Natural Questions (NQ)**, and **HotpotQA** under a strong poisoning setting. It achieves consistently strong detection and localization and substantially reduces attack success while recovering correct answers. The approach is also robust across rerankers (e.g., BGE, mGTE, MS-MARCO MiniLM) and compares favorably to recent defenses.

## Repository structure

> Update the names below to match your repo if needed.

- `src/` — core implementation (feature extraction, MIL model, repair logic, evaluation)
- `scripts/` — runnable entry points (data prep, poisoning, training, evaluation)
- `configs/` — experiment configs (datasets, reranker choice, thresholds, seeds)
- `data/` — dataset splits / indexes (often not fully committed if large)
- `checkpoints/` — trained detector checkpoints (optional; consider release assets)
- `results/` — saved metrics, logs, and plots

## Quickstart (typical workflow)

The end-to-end workflow generally looks like this:

1. **Prepare dataset** (queries + corpus + splits)
2. **Apply poisoning attack** (inject crafted passages for targeted queries)
3. **Retrieve + rerank** and **extract reranker [CLS] features**
4. **Train the MIL detector**
5. **Evaluate detection/localization**
6. **Run repair + generation** and compute attack mitigation metrics

Example commands (replace with your script names/args):

```bash
# Install dependencies
pip install -r requirements.txt

# Prepare data
python scripts/prepare_data.py --dataset nq --out data/nq

# Poison corpus
python scripts/poison_corpus.py --dataset nq --out data/nq_poisoned

# Extract reranker CLS features
python scripts/extract_cls.py --dataset nq --reranker bge --topN 50 --out features/nq_bge

# Train MIL detector
python scripts/train_detector.py --features features/nq_bge --out checkpoints/nq_bge

# Evaluate detection + localization
python scripts/eval_detector.py --ckpt checkpoints/nq_bge --features features/nq_bge

# Repair context + evaluate mitigation
python scripts/eval_repair.py --ckpt checkpoints/nq_bge --dataset nq --k 3
