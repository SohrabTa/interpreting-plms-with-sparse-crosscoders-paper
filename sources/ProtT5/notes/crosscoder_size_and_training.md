# The Target Architecture: ProtT5-XL
Based on the `config.json` for `prot_t5_xl_uniref50`:
- **$d_{model}$ (Residual Stream):** 1,024
- **Encoder Layers ($L_{enc}$):** 24
- **Vocabulary:** Amino acids "vocab_size" : 128 (irrelevant for the SAE size, but relevant for data).

### The Crosscoder Model Size
Unlike a standard SAE which trains on one layer, a **Crosscoder** (as defined in the Anthropic Article) projects from *multiple* source layers into a single shared latent space, and then decodes back to those same layers.

We will assume a standard expansion factor ($F$) of **8** (standard for high-quality interpretability features).

**The Architecture:**
- **Input:** Concatenated activations are *not* used. We learn layer-specific Encoder Weights that sum into the latent space.
- **Latent Space:** $d_{dict} = F \times d_{model} = 8 \times 1,024 = \mathbf{8,192}$.
- **Encoder Weights ($W_{enc}$):** We need a projection matrix for *each* of the 24 layers.
    -   Shape: $[24, 1024, 8192]$
- **Decoder Weights ($W_{dec}$):** We need a reconstruction matrix for *each* of the 24 layers.
    -   Shape: $[24, 8192, 1024]$
- **Biases:**
    -   Encoder Bias: $[8192]$ (Shared)
    -   Decoder Bias: $[24, 1024]$ (Layer-specific)

**Parameter Calculation:**
$$ \text{Params} \approx 2 \times (L \times d_{model} \times d_{dict}) $$
$$ \text{Params} \approx 2 \times (24 \times 1,024 \times 8,192) $$
$$ \text{Params} \approx 2 \times (201,326,592) \approx \mathbf{403 \text{ Million Parameters}} $$

### Training Compute (FLOPs) Estimation
FLOPs for training this Crosscoder on **1 million samples**.

**A. Data Volume (Tokens)**
Let’s assume a context window of **512 tokens** to be safe (padding included or long proteins truncated).
-   Total Tokens ($T$): $1,000,000 \times 512 = \mathbf{5.12 \times 10^8 \text{ tokens}}$ (approx 512M tokens).

**B. FLOPs per Token**
We need to run a forward and backward pass for the Crosscoder.
-   **Forward Pass:**
    -   Encoder (All layers project to latent): $L \times 2 \times d_{model} \times d_{dict}$
    -   Decoder (Latent reconstructs all layers): $L \times 2 \times d_{dict} \times d_{model}$
    -   Total Forward MACs: $4 \times 24 \times 1024 \times 8192 \approx 0.8 \times 10^9$ FLOPs.
-   **Backward Pass:** Rule of thumb is $\approx 2 \times$ Forward Pass.
-   **Total FLOPs per Token:** $\approx 2.4 \times 10^9$ FLOPs.

**C. Total Training FLOPs**
$$ 5.12 \times 10^8 \text{ tokens} \times 2.4 \times 10^9 \text{ FLOPs/token} \approx \mathbf{1.23 \times 10^{18} \text{ FLOPs}} $$

**1.23 ExaFLOPs** in perspective:
-   On a single **NVIDIA A100 (80GB)** getting reasonable utilization (assume ~150 TFLOPS BF16 effectively):
-   $1.23 \times 10^{18} / (150 \times 10^{12}) \approx 8,200 \text{ seconds}$.
-   **Time:** **~2.3 hours on 1 A100.**

### Bottleneck: Activation Storage / Bandwidth

To train the Crosscoder, we need the activations from *all 24 layers* of ProtT5.
-   **Size per token:** $24 \text{ layers} \times 1024 \text{ dim} \times 2 \text{ bytes (BF16)} = 49 \text{ KB per token}$.
-   **Total Dataset Size:** $512 \text{ Million tokens} \times 49 \text{ KB} \approx \mathbf{25 \text{ Terabytes}}$.

We cannot pre-compute and shuffle this dataset. We do not have 25TB of RAM or fast NVMe storage for this.

**Recommendation: Online Training**.
1.  Load ProtT5-XL and Crosscoder onto the GPUs.
2.  Run a batch of proteins through ProtT5.
3.  Harvest activations from all 24 layers from GPU memory.
4.  Pass them directly to the Crosscoder.
5.  Backpropagate through the Crosscoder.
6.  Discard activations.