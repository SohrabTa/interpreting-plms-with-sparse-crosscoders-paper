# Methods:

## SAE Training
### Dataset:
- Follow InterPLM
- 10 million random protein sequences from UniRef50 with length $\leq 1022$
- Sharded into 1000 sequences
- Total of 2.5 billion tokens
- Activations during embedding generation are stored as tensors representing entire shards

### Architecture and Training:
- **Models Interpreted**: ESM2-3B (layers 18 and 36) and ESM2-8M (for comparison)
- **SAE Models**: 
    - 8x expansion factor on the hidden dimension of the ESM2-3B models
    - 16x expansion factor on the hidden dimension of the ESM2-8M models
    - 20,480 latents (dictionary size)
- For all SAE implementations use the dictionary learning library from [Marks et al. (2024)](https://github.com/saprmarks/dictionary_learning)
- **Standard SAE**:
    - Encodes token embeddings $x \in \mathbb{R}^d$ into sparse latent $z \in \mathbb{R}^n$ ($d \ll n$)
    - Decodes to reconstruct $x$: $\hat{x} = W_{dec} z + b_{dec}$
    - **TopK SAE** (Primary): Loss = $\|x - \hat{x}\|_2^2$ (sparsity via hard TopK constraint, $k=100$ for main analysis) (follow [Scaling and evaluating sparse autoencoders by Gao et. al 2024](https://arxiv.org/pdf/2406.04093))
    - **L1 SAE** (Baseline): Loss = $\|x - \hat{x}\|_2^2 + \lambda \|z\|_1$ (follow [Circuits Updates - April 2024 by Conerly et al. 2024](https://transformer-circuits.pub/2024/april-update/index.html))
- **Normalization**: Normalize embeddings to have average unit L2 norm during training. This enables hyperparameter transfer between layers; biases scaled at inference (follow [Marks et al. 2024](https://github.com/saprmarks/dictionary_learning))
- **Optimization**: Use Litdata's optimize function to iterate through each tensor and split it into fixed-size batches -> reduces streaming I/O overhead
- **Data Loading**: Use Litdata's StreamingDataset and StreamingDataloader to shuffle the batches during training
- **Multi-GPU Training**: Use PyTorch Lightning for multi-GPU training

### Evaluation:
- Evaluate SAEs on language modeling and structure prediction tasks

#### Language Modeling:
- Report the average difference in cross-entropy loss $\Delta$CE between the logits from the original and SAE reconstructed PLM. 
- Evaluate on a heldout test set of 10k sequences randomly sampled from Uniref50 on layer 18 of ESM2.

#### Structure Prediction:
- Replace ESM2 hidden representations with SAE reconstructions.
- Feed reconstructed representations into ESMFold structure module.
- Measure $\Delta$ RMSD between ESMFold predictions with original vs reconstructed embeddings.
- **Sparsity Sweep**: Showed reasonable reconstruction with only ~25 active latents (using Matryoshka SAEs).

## Matryoshka SAEs
### Concept:
- Learns hierarchically organized features by forcing nested groups of latents to reconstruct inputs independently.
- Inspired by Matryoshka Embeddings, adapted for SAEs.
- Leads to SAEs that contain features at different levels of abstraction or granularity.

### Architecture:
- **Encoding**:
    - $z = \text{BatchTopK}(W_{enc} x + b_{enc})$
    - BatchTopK keeps only the $B \times K$ highest activations in each batch (enforces average sparsity).
- **Decoding**:
    - Group-wise decoding: Each group $m$ must reconstruct input using only its subset of latents.
    - $\hat{x}^{(m)} = W_{dec}[:, :m] z[:m] + b_{dec}$
- **Loss**:
    - Sum of reconstruction errors across all group sizes $M$:
    - $L(x) = \sum_{m \in M} \|x - \hat{x}^{(m)}\|_2^2 + \alpha L_{aux}$
    - Auxiliary loss ($L_{aux}$) used to reduce dead latents.
- **Groups**: Nested groups of increasing size (e.g., fractions [0.002, 0.0156, 0.125, 0.8574] listed in ESMFold steering study).

## Swiss-Prot Concept Evaluation
- Follow InterPLM by Simon and Zhou (2024) for the Swiss-Prot Concept Evaluation
### Dataset:
- Swiss-Prot manually curated annotations (domains, motifs, binding sites, etc.)

### Method:
- **Modified F1 Score**:
    - Handles mismatch between precise feature activations and broader annotations.
    - **Precision**: Calculated at amino acid level.
    - **Recall**: Calculated at domain level.
- **Comparison**:
    - Evaluated 3B vs 8M models and Matryoshka vs TopK architectures.
    - 3B models significantly outperform 8M models.
    - Matryoshka achieves comparable performance to standard TopK.

## Structure Prediction
### Dataset:
- **CASP14**: Filtered to exclude sequences > 700 residues (for single-GPU efficiency) and targets where ESMFold RMSD > 10 Å.
- Final set: 17 targets.

### Method:
- Replace ESM2 hidden representations with SAE reconstructions.
- Feed reconstructed representations into ESMFold structure module.
- Measure $\Delta$ RMSD between ESMFold predictions with original vs reconstructed embeddings.
- **Sparsity Sweep**: Showed reasonable reconstruction with only ~25 active latents (using Matryoshka SAEs).

## Contact Map Prediction
### Method:
- Follow [Protein language models learn evolutionary statistics of interacting sequence motifs from (Zhang et al. (2024))](https://www.pnas.org/doi/epdf/10.1073/pnas.2406285121)
- Categorical Jacobian calculation: Computes sensitivity of model outputs to mutations to extract coevolutionary signals.
- Applied to SAE reconstructions to test preservation of coevolutionary information.

### Dataset:
- 50 randomly sampled proteins from Zhang et al. evaluation set.

### Hyperparameters:
- **Layer**: 36
- **Architecture**: TopK (Matryoshka skipped due to computational cost and similar performance)
- **ESM2-8M**: 16x expansion (10,240 latents)
- **ESM2-3B**: 8x expansion (20,480 latents)

## ESMFold Steering Case Study
- Adapt from [Scaling monosemanticity by Templeton et al. (2024)](https://transformer-circuits.pub/2024/scaling-monosemanticity)
### Goal:
- Increase structure solvent accessibility (SASA) while keeping sequence fixed.

### Feature Selection:
- Identified a feature (f/2) in the first group of a Matryoshka SAE (layer 36) strongly correlated with residue hydrophobicity.

### Method:
- **Intervention**:
    - Add scaled decoder vector to hidden state: $h_l \leftarrow h_l + \alpha \cdot d(i)$
    - $\alpha'$ (effective coefficient) = norm factor $\cdot \alpha$
        - Accounts for maximum activation scaling applied during training and bias scaling at inference.
        - Ensures consistent steering effects across different model configurations.
- **Ablation**:
    - Isolated intervention effects by using modified layer 36 representation while ablating information from other layers (passing zeros or mean).
- **Result**:
    - Steering increased SASA and RMSD as expected.
    - Demonstrated causal link between specific SAE feature and structural property.
