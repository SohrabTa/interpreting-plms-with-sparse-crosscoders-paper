[[Open Questions - InterPLM]]
# Methods:
## SAE training
### Dataset:
- 5 million random protein sequences from UniRef50 (part of the training dataset for ESM-2)
- Use ESM model and weights from HuggingFace Transformers v.0.24
- For each protein (excluding < cls > and < eos > tokens):
	- extract hidden representations after layers:
		- ESM-2-8M-UR50D: After transformer block layers 1 - 6
		- ESM-2-650M-UR50D: After transformer block layers 1, 9, 18, 24, 30, and 33
- Extracted hidden representation datasets each shareded into groups of 1,000
- tokens shuffled within these groups

### Architecture and Training:
- Pytorch v2.1
- Following Bricken et al. 2023 “Towards monosemanticity”
- For each layer: train 20 SAEs with batch size 2,048
- ESM-2-8M-UR50D:
    - 32x expansion factor (expand 320 to 10,240 features)
    - SAEs for trained for 500,000 steps
    - LR 10^-4 to 10^-8 in 10x increments
    - L1 penalty 0.07 - 0.2
- ESM-2-650M-UR50D:
    - 8x expansion factor (expand 1280 to 10,240 features)
    - SAEs for trained for 200,000 steps
    - LR  5x10^-4 to 5x10^-6 in 5x increments (due to computational constraints)
    - L1 penalty 0.04 - 0.2
- LR and L1 penalty increased linearly from 0
- LR reaching max. within first 5% of steps

### Feature Normalization:
- Research Q1 in [[Open Questions - InterPLM]]

## Swiss-prot concept evaluation pipeline
### Dataset construction
- Randomly sample 50,000 proteins with length <= 1,024 AAs
- Convert all binary and categorical protein-level annotations into binary amino acid-level annotations (how? Q2 in [[Open Questions - InterPLM]])
- Maintain domain-level relationships for multi-amino-acid annotations
- Split this dataset into validation and test sets of 25,000 proteins each. (why? Q3 in [[Open Questions - InterPLM]])
- Only retain concepts present in either:
	- More than ten unique domains (what's a unique domain? Q4 in [[Open Questions - InterPLM]]) OR
	- More than 1,500 amino acids within the validation set
- This resulted in 135 concepts
- Extract concepts from list of annotations listed in Supplementary Table 3
### Feature-concept association analysis
- For each normalized feature:
	- Create binary feature-on/feature-off labels using activation thresholds of 0, 0.15, 0.5, 0.6 and 0.8.
	- Evaluate feature-concept associations using modified classification metrics:
		- $$ \text{Precision} = \frac{\text{true positives}}{\text{true positives} + \text {false positives}}$$
		- $$ \text{Recall} = \frac{\text{domains with true positive}}{\text{total domains}}$$
		- $$ \text{F1} = 2 \times\frac{\text{precision} \times \text{recall}}{\text{precision} + \text{recall}}$$
	- What's a true positive in a feature-concept association? (Q5 in [[Open Questions - InterPLM]])
	- For each feature-concept pair, select the threshold yielding the highest F1 score
### Model selection and evaluation
- Initial evaluations conducted on 20% of the validation set, to compare hyperparameter configurations.
- Only the 135 concepts retained after filtering by number of domains or amino acids they activate for 
- For each of these concepts:
	- identify feature with highest F1 score
	- use average of these top scores to select the best model per layer (the model with the highest average F1 score across all features?)
- Use these six models (one per layer) for the subsequent analyses
- To calculate test metrics:
	- 1. identify features with highest F1 score per-concept on full validation set
	- 2. For each concept:
		- 2.1 calculate F1 score of selected feature on test set
	- 3. Identify all feature-concept pairs with F1 > 0.5 in validation set
	- 4. From those: Get F1 scores on test set and report the ones with F1 > 0.5
### Baselines
- Train randomized baseline models:
	- shuffle values within each weight and bias of ESM2-8M (shuffle weight with bias? Q6 in [[Open Questions - InterPLM]])
	- calculate embedding on same datasets
	- repeat same training (six hyperparameter choices per layer)
		- did they do the same in the for the real SAEs? Q7 in [[Open Questions - InterPLM]]
	- concept-based model selection and metric calculation
- Compare with neurons (Research what that means in Q8 in [[Open Questions - InterPLM]]):
	- Normalize all neuron values between 0 and 1
	- Input into an SAE with expansion factor of 1x
	- identity matrix for encoder and decoder

## LLM feature annotation pipeline
### Example selection
- Random selection of 1,240 (10%) features
- For each feature:
	- Scan 50,000 Swiss-Prot proteins
	- Find those with maximum activation levels in distinct ranges
	- Activation levels quantized into bins of 0.1 (0-0.1, 0.1-0.2, ..., 0.9-1.0)
	- Select two proteins per bin that achieved their max activation in that range
	- Highest bin (0.9-1.0) received ten proteins
	- Include ten random proteins with zero activation as negative examples
	- if fewer than 20 proteins are in the highest bin (0.9-1.0), samples additional examples from second-highest bin (0.8-0.9) until a total of 24 proteins between the two highest bins is reached
	- split them evenly between training and evaluation sets
	- if fewer than 20 proteins are in the top three bins, the feature is excluded
### Description generation and validation
- Create table for each feature containing:
	- protein metadata
	- quantized max activation values
	- indices of activated amino acids
	- amino acid identities at these positions
- Prompt Claude-3.5 Sonnet (new) using this data to generate:
	- detailed description of the feature
	- one-sentence summary
- Instruct the model to generate these descriptions, so they are able to guide activation level prediction for new proteins.
- Validate these descriptions:
	- Provide Claude with an independent set of proteins (from where? Q9 in [[Open Questions - InterPLM]]) and their metadata (matched for size and activation distribution)
	- Claude predicts the activation levels
	- Compare the predicted levels with measure values using Pearson correlation

## Feature analysis and visualization
### UMAP embedding and clustering
- Perform dimensionality reduction on the normalized SAE decoder weights using UMAP with parameters (metric='cosine', neighbors=15, min dist=0.1)
- Perform clustering using HDBSCAN with parameters (min cluster size=5, min samples=3) for visualization

### Sequential and structural feature analysis
- Assess whether SAE features exhibit meaningful spatial organization in protein sequences and structures using the following procedure:
- **Data preparation:**
	- For each SAE feature and each data shard:
		- Use SAE to get feature activation values $a_i$ for each amino acid position $i$ 
		- Identify all positions where activations $a_i > 0.6$ 
		- Retain only proteins with complete AlphaFold 3D coordinate data and at least 25 proteins where activations exceed threshold
		- Randomly sample up to 100 proteins per feature analysis

- **Clustering metric calculation:**
	- For each selected protein:
		- Find position with max activation $\text{pos}_{max} = \text{arg max}_i \, a_i§
		- **Sequential clustering:** 
			- calculate mean activation of sequence neighbours
			- $$ \text{seq\_score} = \frac{1}{\lvert \text{seq\_neighbours}\rvert} \sum_{j \in \text{seq\_neighbours}}{a_j} $$
			- where $\text{seq\_neighbours} = \{j : \lvert j - \text{pos}_{max} \rvert \leq 2, j \neq \text{pos}_{max} \}$ 
		- **Structural clustering:**
			- calculate mean activation of 3D spatial neighbors
			- $$ \text{struct\_score} = \frac{1}{\lvert \text{struct\_neighbours}\rvert} \sum_{j \in \text{struct\_neighbours}}{a_j} $$
			- where $\text{struct\_neighbours} = \{j : \text{distance}(j,\text{pos}_{max}) \leq 6 \, \mathring{A}, j \neq \text{pos}_{max} \}$ (Q10 in [[Open Questions - InterPLM]])
			- Distance calculated using $C_a$ coordinates: $\text{distance}(i,j) = \lvert \lvert C_a(i) - C_a(j) \rvert \rvert$

- **Null distribution generation:**
	- For each protein:
		- Create five random permutations of the activation values $(a_1, a_2, ..., a_n)$
		- For each permutation:
			- Recalculate $\text{seq\_score}_{null}$  and $\text{struct\_score}_{null}$ using the same neighbor positions but shuffled activation values
		- Average $\{\text{seq\_score}_{null}\}$ across the five permutations to get final null values for each protein. (addition: most likely also $\{\text{struct\_score}_{null}\}$ )
- **Statistical testing:**
	- For each feature:
		- Collect observed scores $\{\text{seq\_score}_{observed}\}$ and null scores $\{\text{seq\_score}_{null}\}$ across all proteins
		- Perform paired t-test comparing observed versus null distributions
		- Calculate Cohen's $d$ effect size: $d = \frac{\text{mean(observed - null)}}{\text{s.d.(observed - null)}}$ 
		- Repeat independently for structural clustering scores
- **Multiple testing correction:**
	- Apply Bonferroni correction by multiplying all P values by the total number of features tested per layer (What P values? Q11 in [[Open Questions - InterPLM]])

- Perform this sequential and structural feature analysis separately for each SAE layer
- -> Yields statistical significance and effect size measures for both sequential and structural clustering tendencies of learned features.
- For structural feature identification consider only proteins with Bonferroni-corrected structural $P$ values < 0.05]
- Color features based on the ratio of structural to sequential effect sizes
- Structural-only features are defined as having structural $P$ value < 0.05 but sequential $P$ value > 0.05

### Steering experiments
- Follow approach in "Scaling monosemanticity":
	- 1. Extract embeddings at specified layer
	- 2. Calculate SAE reconstruction and error terms
	- 3. Modify the reconstruction by clamping specified features to desired values
	- 4. Combine modified reconstructions with error terms
	- 5. Allow normal model processing to continue
	- 6. Extract logits and calculate softmax probs for comparisons across steering conditions.
- Steering experiments conducted using NNsight.

### Categorizing Swiss-Prot concepts
- Goal: Analyze concept distributions across models -> developed hierarchical categorization system for the 258 Swiss-Prot concepts identified
- Each concept already has a broad type (e.g. Domain) and a specific subtype (e.g. DRBM)
- Use Claude-3.5 Sonnet to generate detailed functional descriptions for each concept (e.g. 'Domain DRBM' -> 'Double-stranded RNA binding motif')
- Use descriptions to prompt Claude to create biologically meaningful categories and assign each concept to its most appropriate category
- Results in 14 high-level functional categories (e.g. 'Nucleic Acid Interaction Domain') that can be used to compare concept distributions between different-sized pLMs

# Data availability
- Sequences sourced from UniRef50 and Swiss-Prot
- Structures retrieved from AlphaFold DB using version 4 and isoform 1 of each protein (AFDB-F1-v4)
- Per-layer analysis results are available at https://interplm.ai/