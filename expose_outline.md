
# Exposé Draft: Disentangling and Interpreting Protein Language Models with Sparse Crosscoders

## Title
**Disentangling and Interpreting the Hidden Representations of ProtT5 using Sparse Crosscoders**

## Abstract
*Heavily abbreviated, synoptic statement.*
- **Research Need:** Protein Language Models (PLMs) like ProtT5 (trained on BFD and UniRef50) drive state-of-the-art performance in biology but remain opaque "black boxes." While their embeddings are useful, the rich biological knowledge they have likely acquired—such as rules governing stability, binding, and function—remains locked within their weights.
- **Research Question:** Can Sparse Crosscoders effectively disentangle interpretable features across **all layers** of the ProtT5 encoder to provide mechanistic insight?
- **Method:** Train a Sparse Crosscoder on the hidden states of all layers of the ProtT5 encoder. This architecture learns a shared dictionary of features, allowing us to interpret how biological concepts are represented and manipulated throughout the deep network.
- **Expected Result:** A set of interpretable features that can be used to understand the model's internal logic.
- **Benefits:** Bridging the gap between deep learning and mechanistic biology. Crucially, these interpretable features could serve as guiding constraints for "directed evolution" or mutations—enabling a form of **restricted wet-lab in silico**.

## Introduction
- **Topic:** Mechanistic interpretability of large Protein Language Models (PLMs).
- **Focus:** The encoder of ProtT5 (fine-tuned for downstream tasks), specifically analyzing **all layers**.
- **Relevance/Problem:** PLMs capture evolutionary patterns, but standard linear probes often fail to disentangle causal features from correlated confounds. Sparse Autoencoders (SAEs) have shown promise (InterPLM, SAEFold) in isolating distinct features.
- **Specific Gap:** Training independent SAEs for every layer is inefficient and doesn't explicitly model the shared nature of features across depth.
- **Approach:** Apply **Sparse Crosscoders** (introduced by Anthropic). By training a single shared dictionary across all layers, we can interpret features more robustly. We aim to use these features not just for observation, but to guide inputs—enabling interpretable steerability for biological design (directed evolution).

## State of the Art of Research
- **PLMs:** ProtT5 (trained on BFD + UniRef50) is a standard for per-residue prediction and embedding generation.
- **SAEs in Biology:**
    -   *InterPLM (Simon & Zou, 2024)* & *InterProt*: Applied standard SAEs to ESM-2, validating that latent directions correspond to active sites, domains, and functional properties.
    -   *SAEFold*: Applied to structure prediction models.
- **Crosscoders (The Novelty):**
    -   Recent work by Anthropic ("Sparse Crosscoders") demonstrates superior efficiency and interpretability for cross-layer analysis compared to independent SAEs.
    -   **Novelty:** To our knowledge, Sparse Crosscoders have **not yet been applied to any PLM**. This project would be the first to transfer this architecture to the protein domain.

## Methods
- **Model:** ProtT5-XL-U50 (Encoder only). We will extract and analyze hidden states from **all layers**.
- **Architecture:** **Sparse Crosscoder**.
    -   *Input:* Hidden states from all $N$ layers defined as a unified input vector or batched appropriately.
    -   *Loss Function:* We will use the **L2-of-norms** loss. Unlike the L1 version used for direct baseline comparisons, utilizing L2 more efficiently optimizes the frontier of MSE and global sparsity, encouraging the model to find the most efficient shared features without explicit constraints to match per-layer SAE formulations.
    -   *State:* We will target the residual stream or full layer states, following the protocol of reference crosscoder literature.
- **Dataset:**
    -   **Target:** ~10 million protein sequences primarily from **UniRef50**, potentially augmented with a significant portion of **BFD** (Big Fantastic Database).
    -   **Rationale:** This mix aims to mimic the original pre-training data distribution of ProtT5 (which used BFD for pre-training and UniRef50 for fine-tuning), ensuring the features we discover are "native" to the model's learned representation.
- **Analysis & Evaluation:**
    -   **Automated Interpretation:** Use LLMs to annotate features based on maximizing sequences.
    -   **Biological Validation:** Cross-reference active features with UniProt annotations.
    -   **In-Silico Directed Evolution (ProteusAI Integration):**
        -   We will explore using the Crosscoder as a **surrogate model** within a ProteusAI-style evolutionary loop.
        -   Instead of a traditional surrogate classifier predicting fitness, our Crosscoder can directly measure the activation of desirable "feature circuits" (e.g., binding site integrity) to guide the selection of best-fitting protein sequences during evolution.

## Preliminary Work
- **Infrastructure:**
    -   Setup of the core `crosscoder` training codebase (based on Anthropic/open-source implementations).
    -   Configuration of ProtT5 inference pipelines for large-scale hidden state extraction.

## Work Plan and Time Schedule
*(3-Month Guided Research Project)*

- **Month 1: Implementation & Data**
    -   Prepare the training dataset (UniRef50 and possibly BFD).
    -   Extract hidden states from all layers of ProtT5.
    -   Implement the Sparse Crosscoder architecture with L2-of-norms loss.
- **Month 2: Training & Interpretation**
    -   Train the Crosscoder (tuning expansion factor and sparsity penalty).
    -   Generate automated interpretations and map to biological databases.
- **Month 3: Application & Reporting (Optional Paths)**
    -   *Option A (Directed Evolution):* Investigate using features to manually guide specific mutations ("restricted wet-lab in silico").
    -   *Option B (ProteusAI):* Implement the Crosscoder as a surrogate model in ProteusAI to drive automated sequence optimization.
    -   Finalize the expose/report.

## Literature (References)
1.  *Elnaggar, A. et al. (2021). ProtTrans...*
2.  *Simon, N., & Zou, J. (2024). InterPLM...*
3.  *Adams, et al. (2025). InterProt...*
4.  *Parsan, et al. (2025). SAEFold...*
5.  *Anthropic (2024). Sparse Crosscoders for Cross-Layer Features...*
