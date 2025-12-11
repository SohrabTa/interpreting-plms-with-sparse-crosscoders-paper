# Methods:
## SAE training
### Dataset:
- 1 million random protein sequences from UniRef50
- Length restriction: under 1022 residues

### Architecture and Training:
- Model: ESM-2 650M (different layers analyzed, e.g., 8, 16, 24, 28)
- Architecture: TopK Sparse Autoencoder (TopK SAE)
    - Encoder: $z = \text{TopK}(W_{enc}(x - b_{pre}))$
    - Decoder: $\hat{x} = W_{dec} z + b_{pre}$
    - TopK activation: Only keeps the $k$ largest latents non-zero.
    - Loss: Reconstruction MSE $\|x - \hat{x}\|_2^2$
- Hyperparameters (based on analysis sections):
    - Hidden dimension (Expansion factor): 4096 (used in main analysis)
    - Sparsity $k$: 64 (used in main analysis)
    - Note: Paper performs sweeps over $k$ and hidden dimension.

## Latent Classification
### Activation Pattern Classification:
- Rule-based classification of latents based on activation across top sequences.
- Categories (defined in Table 2):
    - **Dead Latent**: Never activated.
    - **Not Enough Data**: < 5 sequences activate.
    - **Periodic**: Regular intervals (freq > 50%, >10 activations/seq, median contig < 10).
    - **Point**: Single residue activation (median length 1).
    - **Motif**: Contiguous regions.
        - Short: 1-20 residues.
        - Medium: 20-50 residues.
        - Long: 50-300 residues.
    - **Whole**: > 80% coverage of sequence.
    - **Other**: Does not fit above criteria.

### Family Specificity Classification:
- Goal: Determine if a latent specifically identifies a protein family.
- Dataset: Swiss-Prot sequences (clustered at 30% identity using MMseqs2).
- Method:
    - Identify InterPro family of the sequence with highest activation for the latent.
    - Binary classification task: Predict membership in that family using latent activation.
    - Sweep normalized activation thresholds (0.1 to 0.9).
    - **Family-specific**: If max F1 score > 0.7 on held-out test set.

## Interpretable Probing
### Downstream Tasks:
- **Secondary Structure**: TAPE dataset (3 classes: helix, strand, other). Residue-level.
- **Subcellular Localization**: DeepLoc dataset (10 classes). Protein-level.
- **Thermostability**: Meltome Atlas / FLIP (Regression). Protein-level.
- **Mammalian Cell Expression**: CHO cell expression (Binary classification). Protein-level.

### Probing Method:
- Train linear probes (Linear Classifier, Logistic Regression, Ridge Regression) on:
    - ESM embeddings (Baseline).
    - SAE latents.
- Pooling: Mean-pooling used for protein-level tasks.
- Evaluation: Compare performance of SAE probes vs ESM probes across layers.
- Interpretation: Analyze coefficients of linear probes to identify most predictive latents and visualize them using InterProt.

## Steering
- **Goal**: Validate family-specific latents.
- **Method**:
    - Select family-specific latent.
    - Clamp activation to fixed levels ($0.5\times, 1\times, 1.5\times, 2\times, 4\times$ max value).
    - Reconstruct activation and pass through remainder of ESM model.
    - Analyze sequence changes (using consensus of top activating sequences as reference).
    - Finding: Steering preserves residues with high family-level consensus.

## Visualization Tool (InterProt)
- Web-based tool to visualize latent activations on protein sequences and AlphaFold structures.
- Features: Align most-activating sequences, highlight structure by activation strength.
