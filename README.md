# CEG-RAG: Cross-Encoder Reranker Signals for Robust RAG Defense Against Poisoning Attacks

This repository contains the reference implementation and experimental code for **“Cross-Encoder Reranker Signals for Robust RAG Defense Against Poisoning Attacks,”** which introduces **CEG-RAG (Cross-Encoder Guardian RAG)**—a practical defense framework designed to protect Retrieval-Augmented Generation (RAG) systems against **corpus/knowledge-base poisoning**.

Retrieval-Augmented Generation is widely used to reduce hallucinations by grounding large language model (LLM) outputs in retrieved evidence. However, RAG also expands the attack surface. If an adversary can inject or manipulate content in the retrieval corpus, they can craft passages that are likely to be retrieved and ranked highly, and then use those passages to steer the generated output toward an attacker-chosen target. In these settings, the generator may appear fluent and confident because it is “supported” by retrieved text—yet it is effectively being guided by malicious evidence. CEG-RAG is built to block that pathway by detecting poisoning in the retrieved context and repairing the context *before* the generator sees it.

CEG-RAG leverages a security signal that already exists in many strong RAG pipelines: the internal representations of a **cross-encoder reranker**. Modern RAG systems frequently rerank retrieved candidates with a cross-encoder because it provides much stronger relevance modeling than dense retrieval alone. Most pipelines use the reranker only for its final scalar relevance score. CEG-RAG instead treats the reranker as a rich feature extractor and uses the reranker’s **final-layer pooled `[CLS]` embedding** for each query–passage pair as a representation of the interaction between the query and that passage. These embeddings contain structured information beyond “how relevant is it,” including patterns that can differentiate genuinely supportive evidence from passages that are relevance-shaped but maliciously constructed.

To learn from realistic supervision, CEG-RAG formulates poisoning detection as a **multiple-instance learning (MIL)** problem. For a given query, the retrieved and reranked set of passages forms a **bag** (the context), and each individual chunk is an **instance** within that bag. This formulation enables training using **context-level labels only** (e.g., whether the overall retrieved context is poisoned) while still producing **instance-level scores** that localize suspicious chunks. In other words, the model learns to answer two questions at once: “Is this retrieved context poisoned?” and “Which chunks in this context are likely responsible?”

When the system predicts that poisoning is present, CEG-RAG performs **context repair prior to generation**. Rather than allowing the generator to consume the original top-ranked context, the framework filters out chunks with high predicted poison risk and replaces them with safer alternatives from lower-ranked candidates, while maintaining a fixed context budget. This design prevents malicious evidence from reaching the LLM, and it does so with low overhead. Importantly, the defense does not require additional LLM calls to verify claims or cross-check sources; it operates using signals that are already computed during reranking and a lightweight detector.

The paper evaluates CEG-RAG on standard retrieval and QA benchmarks including **MS-MARCO**, **Natural Questions (NQ)**, and **HotpotQA**, under strong poisoning settings based on established attacks (e.g., PoisonedRAG-style corpus poisoning). The evaluation focuses on three coupled goals: (1) accurate context-level detection at low false-positive rates, (2) effective chunk-level localization of poisoned passages, and (3) meaningful mitigation, measured by reduced attack success and recovered answer correctness after context repair. The experiments also study robustness across different cross-encoder rerankers (including BGE-, mGTE-, and MiniLM-style rerankers) and compare against recent RAG defense baselines, showing that CEG-RAG provides consistently strong mitigation across datasets.

---

## What you will find in this repository

This codebase is organized to support end-to-end experimentation with CEG-RAG. It includes utilities for producing poisoned corpora and running retrieval/reranking pipelines, modules for extracting cross-encoder `[CLS]` features from query–passage pairs, implementations of the MIL-based detector used for detection and localization, and a context repair module that rewrites the final evidence set before generation. The repository also includes evaluation scripts for reporting detection metrics, localization quality, and end-to-end defense effectiveness.

Because exact file names and folders differ across setups, you should treat the structure below as the intended layout. If your fork or branch uses different names, map the descriptions to your local structure.

- **`src/`** contains the core implementation of the feature extraction pipeline, the MIL detector model, and the context repair logic.
- **`scripts/`** contains runnable entry points for preparing data, applying poisoning, extracting features, training detectors, and evaluating results.
- **`configs/`** stores experiment configurations such as dataset choice, reranker choice, thresholds, context budget, and random seeds.
- **`results/`** stores evaluation outputs such as metrics tables and logs (it is recommended to commit summaries rather than large raw artifacts).
- **`checkpoints/`** stores trained detector checkpoints (for large checkpoints, using GitHub Releases is often cleaner than committing them).

---

## How CEG-RAG experiments typically run

A standard experiment follows a consistent flow.

First, the dataset is prepared by selecting queries and associating them with the underlying corpus and any required splits. Next, a corpus poisoning procedure injects crafted passages designed to be retrieved and to influence generation. Retrieval is then performed and candidate passages are reranked with a cross-encoder. For each query–passage pair in the reranked list, the system extracts the reranker’s pooled `[CLS]` embedding and constructs a feature matrix representing the retrieved context. These features are fed into the MIL detector, which outputs a context-level poisoning prediction and instance-level risk scores. If poisoning is detected, the context repair step filters high-risk passages and fills the context with safer candidates while keeping the same context budget. Finally, the generator produces an answer from the repaired context, and the evaluation measures detection and localization performance as well as mitigation effectiveness (attack success reduction and answer recovery).

---

## Key parameters and concepts

Several parameters control the behavior of the defense and are commonly exposed through configuration files.

The number of reranked candidates analyzed per query determines how much context the detector sees and directly affects the size of the feature matrix. The context budget controls how many passages are ultimately provided to the generator, and it is kept fixed during repair to ensure stable generation cost. The MIL grouping strategy (often expressed as a block or bag size) determines how the retrieved list is partitioned for bag-level inference. Finally, two thresholds control the defense decision-making: a detection threshold that decides whether repair is triggered, and a localization threshold that decides which chunks are removed as high-risk during repair.

---

## Getting started (general)

Install dependencies using your preferred environment manager, then run the scripts in the typical order: data preparation, poisoning, feature extraction, training, evaluation, and repair-based mitigation evaluation. The exact commands depend on your local script names and arguments, but the workflow remains the same.

```bash
pip install -r requirements.txt
