
### SAE Model Dimensions
- **Input Dimension ($d_{in}$):** 1,024
- **Expansion Factor:** 8x
- **Hidden Dimension ($d_{hidden}$):** $1,024 \times 8 = 8,192$

### Parameter Calculation (Per SAE Instance)
A standard Sparse Autoencoder (SAE) used for interpretability consists of an encoder (weight + bias) and a decoder (weight + bias). The weights are typically **untied** (separate matrices for encoder and decoder).

- **Encoder Weights ($W_{enc}$):**
  $$d_{in} \times d_{hidden} = 1,024 \times 8,192 = 8,388,608$$
- **Encoder Bias ($b_{enc}$):**
  $$d_{hidden} = 8,192$$
- **Decoder Weights ($W_{dec}$):**
  $$d_{hidden} \times d_{in} = 8,192 \times 1,024 = 8,388,608$$
- **Decoder Bias ($b_{dec}$):**
  $$d_{in} = 1,024$$

**Total Parameters:**
$$8,388,608 + 8,192 + 8,388,608 + 1,024 = \mathbf{16,786,432}$$

### Important Considerations for Layers
- **Single Layer vs. All Layers:** The calculation above is for **one** SAE trained on a single layer's representations.
- **Full Model Coverage:** To train a separate SAE for every layer of the ProtT5 Encoder (which has **24 layers**), the total parameter count for the full suite of SAEs would be:
  $$16.8 \text{ million} \times 24 \approx \mathbf{403 \text{ million parameters}}$$