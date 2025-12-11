[[Notes for "InterPLM discovering interpretable features in protein language models via sparse autoencoders"]]

1. Was the feature normalization applied during training or only during inference? (Methods -> Feature normalization.)
	1. [ChatGPT thinks](https://chatgpt.com/c/690f5a82-9e50-832d-9017-b31471f0b75a) it's a post-training process, where the corresponding encoder and decoder weights of each feature were rescaled.
	2. Try to calculate it on paper and see if that makes sense. Does every latent have a corresponding encoder row and decoder column in the weight matrices?
    
    **Answer:** The feature normalization was applied **post-training** (during the analysis/inference phase). The Methods section ("Feature normalization") states: "To standardize feature comparisons, we normalized activation values using a scan across 50,000 proteins from Swiss-Prot... and used it to scale the encoder weights and decoder weights reciprocally". This scaling acts as a permanent adjustment for the interpretation phase to ensure all features are scaled between 0 and 1.
2. How exactly did they convert all binary and categorical protein-level annotations into binary amino acid-level annotations?

    **Answer:** The paper states they "converted all binary and categorical protein-level annotations into binary amino acid-level annotations, maintaining domain-level relationships for multi-amino-acid annotations" (Methods). However, the quantitative analysis (Supplementary Table 3) primarily used feature-level ("ft") annotations (like domains, active sites, helices) which naturally map to specific amino acid ranges. Protein-level annotations ("cc") like general function were explicitly excluded from the quantitative concept analysis (marked "Quant: N"). For the included annotations, the conversion is a binary mask: 1 for residues within the annotated region (e.g., inside the Domain), 0 otherwise.
3. Why do we need validation and test set for the concept evaluation pipeline? Isn't the concept evaluation pipeline started post-training of SAEs in inference?

    **Answer:** The validation set is used for **model selection** (identifying which SAE feature corresponds best to a given biological concept by maximizing F1 score), while the test set is used for **evaluation** (reporting the F1 score of that selected feature-concept pair on unseen data). This separation is crucial to prevent overfitting the feature selection process—ensuring that the matched feature actually generalizes to detecting the concept in new proteins, rather than just being the best match by chance in the selection pool.
4. What exactly is a domain? Furthermore, what makes a domain unique? 

    **Answer:** In this context, a "domain" refers to "**Distinct functional or structural protein units**" as annotated in Swiss-Prot (field `ft domain`, see Supp Table 3). "Unique domains" likely refers to distinct **types** of domains (e.g. "Kinase domain" vs "SH3 domain") or distinct instances in the dataset depending on context (e.g. "concepts present in more than ten unique domains" likely means 10 distinct occurrences to allow statistical validity). 
5. What's a true positive in a feature-concept association?
	1. They then scan all amino acids across the dataset where the concept is annotated (e.g. binding site residues).
	2. For each residue, they record:
		1. True positives (feature on & concept present)
		2. False positives (feature on & concept absent)
		3. False negatives (feature off & concept present)
	3. Under the assumption that feature $f_i$ is a detector for concept $c_j$  a true positive is, if and the feature and concept are active for the same input amino acid.

    **Answer:** 
    *   **Per-residue Precision (Amino Acid Level):** A true positive is indeed an amino acid where both the feature is active (above threshold) and the concept annotation is present.
    *   **Per-domain Recall (Domain Level):** The paper calculates recall per domain. A "domain with true positive" (Eq 2) is a domain instance that contains at least one true positive amino acid (or sufficiently activated region). The text notes: "because it correctly identifies two amino acids in every domain tested, it now has a domain-adjusted recall of 1.0."
6. Every layer exists of $dim_h$ weights + one bias value, right? I assume it's not like in a fully connected layer, a transformer layer's weights are different. Does "shuffle values within each weight and bias" mean, we just randomly shuffle the $dim_h + 1$ weights and biases?

    **Answer:** No, the user's assumption about layer structure is too simple. A Transformer layer contains multiple large matrices (Attention Q, K, V, O projections; MLP up/down projections; LayerNorm parameters), containing millions of parameters. "Shuffled the values within each weight and bias" means they took **every parameter tensor** in the ESM-2 model and randomly permuted the values within that tensor. This preserves the global statistics (mean, variance) of the weights but completely destroys the learned structure and transformations.
7. They mention in the baseline models, that they tried six hyperparameter choices per layer, did they try the same number for the real SAEs? In the real SAEs they mentioned 20 models per layer.

    **Answer:** No, they did not use the same number. For the **real SAEs**, they trained **20 models per layer** (sweeping L1 penalties). For the **randomized baselines**, they "repeated the same training (using six hyperparameter choices per layer)" (Methods, Baselines). This implies a reduced search space was used for the baselines.
8. Research how the compare with neurons section in the baseline model comparison works in detail and what they investigated with it.

    **Answer:** To compare SAE features with neurons, they treated individual neurons of the original model as if they were SAE features. Specifically, they "scaled all neuron values between 0 and 1... then input these into an SAE with expansion factor of 1× that has an identity matrix for the encoder and decoder" (Methods, Baselines). They then applied the exact same Swiss-Prot concept evaluation pipeline to these "neuron-features". They investigated whether neurons align with biological concepts as well as SAE features do, finding that SAE features achieve significantly higher F1 scores and capture more biological concepts (Results, Fig 2).
9. Where does that independent set of proteins come from? Aren't all of them listed in the feature tables?

    **Answer:** The "independent set of proteins" used for LLM validation is distinct from the examples provided in the prompt. The dataset was split into validation and test sets (25,000 each). The prompt was constructed using specific selected examples (40 proteins, likely from the validation or training split), but for validation, they provided Claude with metadata for *other* proteins ("independent set", likely from the hold-out or test set) and asked it to predict their activations. This tests generalization beyond the specific examples shown in the context window.
10. Whats $\mathring{A}$ ?

    **Answer:** $\mathring{A}$ stands for **Angstrom**, a unit of length equal to $10^{-10}$ meters (0.1 nanometers). It is the standard unit used in structural biology to measure atomic distances and bond lengths. In the paper, it is used to define "structural neighbors" (residues within 6 $\mathring{A}$ of each other).
11. What do they mean by P values?

    **Answer:** The P values refer to the statistical significance of the feature clustering analysis. They calculated a "structural score" (mean activation of 3D neighbors) and compared it to a "null distribution" generated by shuffling activation values 5 times. The P value is derived from a **paired t-test** comparing the observed scores to these null scores, quantifying the probability that the observed structural clustering of the feature is due to chance. They also used P values (Wilcoxon signed rank test) to compare the distribution of benign vs pathogenic mutation effects.