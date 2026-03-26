# How Large Language Models Work

**Executive summary:** Large Language Models (LLMs) are neural networks trained on massive text corpora to model language. Early approaches used n-grams or recurrent nets to predict words sequentially【18†L159-L167】【42†L7-L10】. The transformer architecture (Vaswani *et al.* 2017) introduced self-attention and parallel processing, becoming the foundation of modern LLMs【9†L226-L233】【14†L281-L289】. Training involves massive datasets (Common Crawl, Wikipedia, Books, etc.), tokenization (Byte-Pair Encoding or WordPiece【35†L90-L93】【36†L90-L93】), and objectives like next-token (autoregressive) or masked-language modeling【66†L277-L280】. Scaling laws show that model performance (cross-entropy loss) improves predictably with more parameters, data, and compute following power-laws【39†L114-L122】. After pretraining, models can be fine-tuned on tasks or aligned via instruction-tuning and RLHF (reinforcement learning from human feedback)【41†L61-L69】【41†L67-L72】. At inference, LLMs generate text with decoding strategies (greedy, beam-search, top-*k*/top-*p* sampling)【22†L124-L132】【24†L187-L195】. Evaluation uses metrics like perplexity【42†L7-L10】 or task-specific scores (accuracy, BLEU, etc.). LLMs exhibit failure modes (hallucinations, bias) reflecting training data【46†L843-L851】【46†L860-L863】, so safety measures (data filtering, RLHF, rule-based models) are applied【41†L67-L72】【49†L9-L18】. This report details the history, architecture, training, inference, and evaluation of LLMs, with code snippets and diagrams for clarity.

## 1. Historical context: n-grams, RNNs, attention

Early language models counted sequences of words (**n-grams**) under a **Markov assumption**, e.g. \(P(w_n|w_{1:n-1}) \approx P(w_n|w_{n-1})\) for bigrams【18†L159-L167】. Such models assign probabilities by frequency counts, but suffer data sparsity and cannot capture long-range context. Neural methods replaced this with **recurrent neural networks (RNNs)**, where each token is processed sequentially through hidden states. Vanilla RNNs (and LSTMs/GRUs) learned distributed representations of history and could (in theory) model long context, but in practice are slow to train and suffer from vanishing gradients over long sequences. 

To improve capacity, attention mechanisms were introduced (e.g. **Bahdanau et al.** 2015) in RNN sequence-to-sequence models, allowing the decoder to *focus on* relevant parts of the input. This attention weights encoder states via learned compatibility scores. In **“Attention Is All You Need” (Vaswani et al. 2017)**, the authors abandoned recurrence entirely and built an architecture solely from *self-attention* and feed-forward layers. In a transformer encoder, each token **attends to all others** in the sequence, enabling parallel processing and capturing global context. This innovation underlies modern LLMs. Transformer-like models (GPT, BERT, etc.) quickly surpassed RNNs on benchmarks (GLUE, translation, question answering)【66†L263-L270】【73†L37-L40】.

## 2. Transformer architecture

A **Transformer block** consists of (1) a multi-head self-attention layer, (2) add-and-norm, (3) a feed-forward network (FFN), and (4) another add-and-norm (Figure below). In self-attention, each input token is projected to query \(Q\), key \(K\), and value \(V\) vectors. The attention output is a weighted sum of values: 

\[
\text{Attention}(Q,K,V) = \text{softmax}\Bigl(\tfrac{QK^T}{\sqrt{d_k}}\Bigr)V,
\] 

as given by Vaswani *et al.*【9†L226-L233】. The scaling by \(\sqrt{d_k}\) keeps gradients stable. In practice, queries \(Q\), keys \(K\), and values \(V\) are matrices for all tokens. **Multi-head attention** runs several attention “heads” in parallel: each head uses different learned linear projections \(W_i^Q,W_i^K,W_i^V\) for its own \(Q,K,V\), computes attention, and then the head outputs are concatenated and projected by \(W^O\)【14†L281-L289】. Formally, for \(h\) heads: 

\[
\text{MultiHead}(Q,K,V) = \text{Concat}(\text{head}_1,\dots,\text{head}_h)W^O,\quad \text{head}_i = \text{Attention}(QW_i^Q,\;KW_i^K,\;VW_i^V).
\] 

Multi-head attention allows the model to jointly attend to information from different subspaces and positions【14†L281-L289】. 

After attention, a **residual connection** and layer normalization【14†L281-L289】 produce an intermediate representation. Then a position-wise feed-forward network (two linear layers with a ReLU) transforms each token independently. Another add-and-norm follows. 

To encode token order without recurrence, transformers add **positional encodings** to input embeddings (e.g. sine/cosine functions or learned vectors)【14†L345-L352】. 

```mermaid
flowchart TB
    A[Input Embeddings + Positional Encoding] --> B[Multi-Head Self-Attention]
    B --> C[Add & LayerNorm]
    C --> D[Feed-Forward (2-layer MLP)]
    D --> E[Add & LayerNorm]
    E --> F[Block Output]
```
*Figure: Transformer encoder block. Tokens are processed by multi-head self-attention (with queries, keys, values), followed by residual add-and-norm, a feed-forward network, and another add-and-norm.* 

Each transformer layer has **O(n²d)** compute per layer (with \(n\) = sequence length, \(d\) = hidden size) due to the all-pairs dot-products【9†L226-L233】. By contrast, an RNN layer is O(n·d²) sequentially. This parallel structure makes transformers efficient on modern hardware. In practice, stacks of ~12–96 layers (e.g. 12 for BERT-base, 96 for GPT-4 style models) are used, with hundreds of millions to trillions of parameters.

## 3. Training pipeline

### 3.1 Pretraining data and objectives

LLMs are **pretrained** on vast text corpora. For example, GPT-3 (175B parameters) used a filtered Common Crawl (60% of data, ~300B tokens), plus WebText2 (from Reddit), books, and Wikipedia【31†L215-L224】. Data is deduplicated and filtered for quality【31†L215-L224】. BERT (330M tokens) used Wikipedia and BookCorpus. Tokenization splits text into subword tokens via **Byte-Pair Encoding (BPE)** or **WordPiece**. BPE (used by GPT-2) merges frequent byte sequences into tokens【35†L90-L93】, while WordPiece (used by BERT) similarly builds subwords with a prefix (e.g. “##”)【36†L90-L93】. Both methods produce a vocabulary (typically 30–50k tokens) to balance granularity and coverage. 

Common pretraining objectives are:

- **Autoregressive Language Modeling (next-token prediction):** The model predicts each token given all previous ones. This is used by GPT series, Bloom, etc. The loss is cross-entropy of predicting the next token. (Algorithmically, this is typically a causal (masked) self-attention).  
- **Masked Language Modeling (MLM):** The model masks a random subset of tokens (e.g. 15%) and predicts them from both left and right context. This is used by BERT【66†L277-L280】 (also RoBERTa, DistilBERT). The MLM objective (a “cloze” task) enables bidirectional context.  
- **Seq2Seq / Encoder-Decoder:** Models like T5/BART treat tasks as text-to-text. They may use a prefix token indicating the task and train to output target sequences (an autoregressive decoder given an encoder or prefix). 

These objectives align with different model architectures (decoder-only vs encoder-only vs encoder-decoder). See Table 1 for a summary:

| **Pretraining Objective**      | **Task**                          | **Example Models**          |
|--------------------------------|-----------------------------------|-----------------------------|
| Autoregressive LM (next-token) | Predict \(w_t\) given \(w_{<t}\) (causal) | GPT, GPT-2, GPT-3/4, Bloom |
| Masked LM (MLM)               | Mask random tokens, predict them   | BERT, RoBERTa, DistilBERT   |
| Prefix/Seq2Seq LM            | Generate target from input prefix  | T5, BART, UL2              |

*Table 1: Common pretraining objectives and example models.*  

### 3.2 Compute and scaling laws

Training these models demands enormous computation. For example, GPT-3 required on the order of \(3\times10^{23}\) floating-point operations【72†L467-L470】 (roughly hundreds of GPU-years). Large models obey **scaling laws**: Kaplan *et al.* (2020) showed that validation loss (cross-entropy) follows power-laws in model size \(N\), dataset size \(D\), and compute \(C\)【39†L114-L122】. Larger models and more data reduce loss smoothly. Specifically, for transformer LMs (at optimal data/model scaling), loss \(L\) vs compute \(C\) follows roughly \(L \propto C^{-0.076}\) over many orders of magnitude【39†L114-L122】. This means *every 10× increase in compute yields a fixed fractional drop in loss*. Figure 1 illustrates this trend:

【63†embed_image】 *Figure: Empirical scaling law – validation loss vs compute for increasing model/data/compute (power-law behavior)【39†L114-L122】.*

The scaling laws imply **sample-efficiency**: larger models achieve lower loss with fewer data/updates【39†L146-L151】. They also predict an optimal allocation of a fixed compute budget among \(N\), \(D\), and training steps. Crucially, performance depends most on *scale* (parameters, data, compute) and much less on architectural details like width vs depth【39†L104-L112】. In practice, this drives ever-larger models (GPT-4 scale) trained on internet-scale data.

### 3.3 Tokenization and data pipeline

Before training, raw text is cleaned and tokenized. Common pipelines use libraries like Hugging Face **Tokenizers**【35†L90-L93】. For example:

```python
from transformers import AutoTokenizer
tokenizer = AutoTokenizer.from_pretrained("gpt2")  # byte-level BPE tokenizer
tokens = tokenizer("Hello world!", return_tensors="pt")
print(tokens.input_ids)  # tensor of token IDs
```

For BERT, WordPiece is used【36†L90-L93】: “unbelievable” → [“un”, “##believable”], etc. The pipeline then creates batches of tokenized sequences (fixed max length, padding) for training. 

## 4. Fine-tuning and instruction alignment

After pretraining, LLMs are **fine-tuned** for tasks or to align with desired behaviors. Fine-tuning can be supervised (task-specific data) or rely on reinforcement learning from human feedback (RLHF). 

In RLHF (used in InstructGPT and ChatGPT), one first collects human demonstrations of a “good” answer for prompts, and fine-tunes the model on these (Supervised Fine-Tuning, SFT). Next, human annotators rank multiple model responses per prompt. A *reward model* is trained to predict these rankings. Finally, the LLM is updated via reinforcement learning (e.g. PPO) to maximize the reward. This process aligns the model’s outputs with human preferences. Ouyang *et al.* (2022) showed InstructGPT (aligned GPT-3) preferred by humans and more truthful/toxic-safe than the raw GPT-3【41†L61-L69】【41†L67-L72】. InstructGPT’s 1.3B model outperformed the 175B GPT-3 on human preference despite 100× fewer parameters【41†L67-L72】. This demonstrates RLHF’s impact. 

**RLHF summary:**  
- Collect *demonstrations*: human-written answers for many prompts.  
- Fine-tune pretrained LLM (supervised) on this dataset.  
- Collect *comparisons*: humans rank model outputs for prompts.  
- Train a *reward model* on these rankings.  
- Use policy optimization (RL) to tune the LLM to maximize the reward.  

This yields **instruction-tuned** models that follow user prompts better. Recent LLMs (GPT-4, Claude, etc.) rely on sophisticated variations of RLHF or “constitutional” AI to ensure safer outputs【49†L9-L18】【41†L63-L72】.

## 5. Inference and decoding

At inference/generation time, LLMs produce tokens sequentially using the learned distribution \(P(w_t|w_{<t})\). Various **decoding strategies** can trade off quality and diversity【22†L124-L132】【22†L156-L160】:

- **Greedy search:** Pick the highest-probability next token at each step. Simple but prone to repetitive or suboptimal output【22†L124-L132】.  
- **Beam search:** Keep the top *k* sequences (beams) at each step, expanding all and selecting best *k*. Explores more possibilities, often used in translation but can still lack diversity【24†L187-L195】.  
- **Sampling:** Randomly sample from the token distribution (with `do_sample=True`). Generates more varied output.【22†L156-L160】.  
- **Top-*k* sampling:** Restrict sampling to the *k* most probable tokens at each step (zero out others).  
- **Top-*p* (nucleus) sampling:** Restrict to the smallest set of tokens whose cumulative probability ≥ *p* (e.g. 0.9). This adapts the cutoff dynamically.  
- **Temperature:** Scale logits by 1/*T* before softmax. Higher temperature (*T*>1) flattens the distribution (more randomness); lower *T* sharpens it.  

Hugging Face Transformers makes decoding straightforward. For example (PyTorch): 

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("gpt2")
model = AutoModelForCausalLM.from_pretrained("gpt2")
inputs = tokenizer("The weather today is", return_tensors="pt")
# Greedy decode
greedy_ids = model.generate(**inputs, max_new_tokens=20)
print(tokenizer.decode(greedy_ids[0]))

# Sampling with temperature and nucleus
sample_ids = model.generate(**inputs, max_new_tokens=50, do_sample=True, top_k=50, top_p=0.95, temperature=0.8)
print(tokenizer.decode(sample_ids[0]))
```

Table 2 summarizes decoding methods:

| **Decoding Method**       | **Description**                                            |
|---------------------------|------------------------------------------------------------|
| Greedy                    | Choose highest-probability token each step【22†L124-L132】. Quick but can repeat.  |
| Beam search               | Keep *B* best sequences (beams) each step【24†L187-L195】. Searches more but slower. |
| Top-*k* sampling          | Randomly sample from top *k* tokens (multinomial).         |
| Top-*p* (nucleus) sampling| Sample from smallest set of tokens with cumulative prob ≥ *p*. Avoids tail tokens.  |
| Temperature scaling       | Divide logits by *T* before softmax (higher *T* → more randomness). |

*Table 2: Common decoding strategies (see Hugging Face docs【22†L156-L160】【24†L187-L195】).* 

## 6. Evaluation metrics

LLM performance is evaluated both intrinsically and on downstream tasks. 

- **Perplexity (PPL):** A standard intrinsic metric, defined as \(2^{\text{cross-entropy}}\). Lower perplexity means better predictive performance【42†L7-L10】. It measures how well the model predicts held-out text. For language models, perplexity is the exponential of average negative log-probability. (Jurafsky & Martin define it as the “weighted average branching factor”【42†L31-L39】.) 
- **Downstream benchmarks:** Models are fine-tuned or prompted on tasks (QA, classification, generation) and scored by task-specific metrics. Examples: GLUE/SuperGLUE for understanding (accuracy/F1)【73†L37-L44】, SQuAD for QA (F1/EM), BLEU/ROUGE for translation or summarization, accuracy on common sense tests, etc.  
- **Bias/toxicity metrics:** Specialized tests (e.g. RealToxicityPrompts) measure generation of harmful content. Truthfulness benchmarks (TruthfulQA) evaluate factual accuracy. Hallucination can be measured by comparing generated facts to ground truth (often manually).  

Improvement in perplexity generally correlates with downstream gains【42†L119-L128】, but not perfectly – better modeling (intrinsic) is a necessary but not sufficient condition for task performance. Thus LLMs are evaluated on many tasks and held-out datasets to ensure general competence.

## 7. Common failure modes and biases

Despite their power, LLMs have well-known limitations:

- **Hallucinations / factual errors:** They can produce plausible but incorrect or invented information, since they only model patterns in training text and have no grounded truth-check. For example, given ambiguous prompts, models may “hallucinate” plausible-sounding but false answers. RLHF and retrieval augmentation (e.g. retrieval-augmented generation) are used to mitigate this. 
- **Bias and stereotypes:** Models trained on internet data reflect its biases. Studies (including analysis of GPT-3) show models can generate stereotypical or prejudiced content (gender, race, religion)【46†L843-L851】. For instance, an English model might associate certain professions with a single gender. These biases are “internet-scale” – whatever is common in the data is amplified【46†L843-L851】【46†L860-L863】.  
- **Toxicity/offensiveness:** Unfiltered text corpora contain toxic or offensive language. LLMs may output such language, especially in edge-case prompts.  
- **Poor handling of context limits:** Transformers have a maximum context length (e.g. 2048 tokens, often 8k–32k in modern models). Long documents get truncated or require expensive mechanisms (long-range attention, memory).  
- **Ambiguity and instruction failure:** Without careful prompting, models may misinterpret instructions or produce unhelpful answers. Even instruction-tuned models can break down on adversarial prompts or ambiguity.  
- **Adversarial prompts:** Cleverly crafted inputs (jailbreaks) can bypass safety filters, causing models to output disallowed content or behave undesirably. 

These failure modes arise from **training data gaps and model design**. For example, hallucination partly reflects that maximum likelihood training (or RLHF) does not penalize ungrounded but coherent statements. Bias comes from skewed corpora. 

## 8. Safety and mitigation strategies

To reduce risks, practitioners employ multiple strategies:

- **Data curation/filtering:** Remove or downweight toxic content, personal data, or copyrighted text from pretraining corpora. For instance, GPT-3 filtered Common Crawl to high-quality text【31†L215-L224】.  
- **Reinforcement learning from human feedback (RLHF):** As noted, this aligns models with human values and reduces undesirable outputs. InstructGPT showed lower toxicity and more truthful answers while being similarly fluent【41†L67-L72】.  
- **Rule-based or AI-based filters:** Systems like OpenAI’s moderation API or rule-based “constitutional” AI frameworks (e.g. Levy et al., 2022) post-process outputs to flag or edit harmful content. Recent work (Rule-Based Rewards) even uses large language models as feedback to shape safe behavior【49†L9-L18】.  
- **Adversarial testing:** Red teams intentionally probe models for harmful behavior; the findings inform additional fine-tuning or prompt engineering.  
- **Ongoing alignment research:** Including efforts in “safe completion” techniques, continual fine-tuning on new safety data, and transparency measures. 

No mitigation is perfect. The field treats alignment as ongoing work. Nevertheless, combining content filtering, RLHF, and careful monitoring is the current best practice for reducing harmful outputs.

## 9. Practical usage (code snippets)

Here we illustrate basic usage with Hugging Face Transformers (PyTorch). First, a forward pass through a pretrained causal LM (GPT-2):

```python
from transformers import AutoTokenizer, AutoModelForCausalLM
import torch

tokenizer = AutoTokenizer.from_pretrained("gpt2")
model = AutoModelForCausalLM.from_pretrained("gpt2")

text = "Artificial intelligence is"
inputs = tokenizer(text, return_tensors="pt")
outputs = model(**inputs)      # forward pass
logits = outputs.logits       # shape [1, seq_len, vocab_size]

print("Logits shape:", logits.shape)
```

The `logits` give predicted scores for the next token at each position. To generate text:

```python
gen_ids = model.generate(
    **inputs,
    max_new_tokens=50,
    do_sample=True,
    top_k=50,
    top_p=0.95,
    temperature=0.8
)
generated = tokenizer.decode(gen_ids[0], skip_special_tokens=True)
print("Generated:", generated)
```

This uses *nucleus sampling* with temperature. The `generate()` API supports all strategies mentioned above (see Hugging Face docs【22†L156-L160】). 

## 10. Complexity and compute estimates

Transformer layers scale as \(O(n^2d)\) per layer, where \(n\) is sequence length and \(d\) hidden size. This limits context length due to memory/time. Training costs grow as \(O(N \cdot S \cdot d^2)\) roughly, where \(N\) is parameters and \(S\) steps. For example, training GPT-3 (~175B parameters) took on the order of \(3\times10^{23}\) floating-point operations【72†L467-L470】. By scaling laws【39†L114-L122】, even larger models require exponentially more compute (GPT-4 likely exceeded \(10^{25}\) FLOPs). Inference cost per token is much lower: roughly a few hundred FLOPs per parameter. For instance, a 100B-parameter model needs on the order of \(10^{13}\) FLOPs to generate 100 tokens. 

Memory-wise, storing \(n\) tokens’ activations (for backprop) also scales with \(O(n d)\) per layer. This is why sequence lengths are capped (e.g. 2048) on many models, although techniques like sparse attention or chunking can extend this. 

## 11. Future directions

LLMs continue evolving. **Efficiency:** Research on sparse and mixture-of-experts architectures (e.g. Switch Transformers) aims to reduce compute per token. **Long-context models:** New attention variants (Linformer, Performer) or recurrent memory augmentations aim to handle much longer contexts. **Multimodality:** Models like PaLM-E or GPT-4V integrate vision or other modalities with language. **Retrieval integration:** Retrieval-augmented LMs (e.g. RETRO, RAG) combine a frozen LLM with an external database to reduce hallucination. **Finer alignment:** Continued work on safer RLHF, “constitutional AI”, and grounding with real-world data. 

Open problems include truly safe alignment, understanding model internals, and reducing carbon footprint. However, given scaling trends and community focus, LLMs will likely get more capable and specialized (e.g. domain-tuned), while research seeks to make them more reliable and controllable. 

**Assumptions:** This overview assumes standard transformer-based LLMs with sequence lengths ≤8k, training on common datasets (Web, Books, Wikipedia), and modern compute hardware (GPUs/TPUs). Specific numbers (parameter counts, flop budgets) vary by model generation, but we cite representative values. 

## References

- Vaswani *et al.*, “Attention Is All You Need,” NeurIPS 2017【9†L226-L233】【14†L281-L289】.  
- Devlin *et al.*, “BERT: Pre-training of Deep Bidirectional Transformers,” ACL 2019【66†L277-L280】.  
- Brown *et al.*, “Language Models are Few-Shot Learners” (GPT-3 NeurIPS 2020)【31†L215-L224】.  
- Kaplan *et al.*, “Scaling Laws for Neural Language Models,” arXiv 2020【39†L114-L122】【39†L104-L112】.  
- Ouyang *et al.*, “Training language models to follow instructions with human feedback” (InstructGPT, NeurIPS 2022)【41†L61-L69】【41†L67-L72】.  
- Hugging Face Transformers documentation (Decoding strategies)【22†L124-L132】【22†L156-L160】【24†L187-L195】.  
- Jurafsky & Martin, *Speech and Language Processing* (3rd ed.), “Perplexity”【42†L7-L10】.  
- Jurafsky & Martin, *Speech and Language Processing*, “The Markov assumption / n-grams”【18†L159-L167】.  
- W. Zaremba *et al.* (OpenAI blogs and model docs on model release and scaling).  
- Mu *et al.*, “Rule Based Rewards for Language Model Safety,” OpenAI technical report 2023【49†L9-L18】.  
- Schick *et al.*, *AI Capabilities vs Compute* (analysis of GPT-3 flops)【72†L467-L470】.  

