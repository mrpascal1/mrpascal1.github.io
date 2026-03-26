---
layout: post
author: Shahid Raza
title: How Large Language Models Work
tags: Engineering ML AI GPT LLM
---

Large Language Models (LLMs) are neural networks trained on massive text corpora to model language. Early approaches used n-grams or recurrent nets to predict words sequentially. The transformer architecture (Vaswani *et al.* 2017) introduced self-attention and parallel processing, becoming the foundation of modern LLMs. Training involves massive datasets (Common Crawl, Wikipedia, Books, etc.), tokenization (Byte-Pair Encoding or WordPiece), and objectives like next-token (autoregressive) or masked-language modeling. Scaling laws show that model performance improves predictably with more parameters, data, and compute following power-laws. After pretraining, models can be fine-tuned on tasks or aligned via instruction-tuning and RLHF (reinforcement learning from human feedback). At inference, LLMs generate text with decoding strategies (greedy, beam-search, top-*k*/top-*p* sampling). LLMs exhibit failure modes (hallucinations, bias) reflecting training data, so safety measures (data filtering, RLHF, rule-based models) are applied. This post details the history, architecture, training, inference, and evaluation of LLMs, with code snippets and diagrams for clarity.

---

## 1. Historical context: n-grams, RNNs, attention

Early language models counted sequences of words (**n-grams**) under a **Markov assumption**, e.g. $P(w_n \mid w_{1:n-1}) \approx P(w_n \mid w_{n-1})$ for bigrams. Such models assign probabilities by frequency counts, but suffer from data sparsity and cannot capture long-range context.

Neural methods replaced this with **recurrent neural networks (RNNs)**, where each token is processed sequentially through hidden states. Vanilla RNNs (and LSTMs/GRUs) learned distributed representations of history and could (in theory) model long context, but in practice are slow to train and suffer from vanishing gradients over long sequences.

To improve capacity, attention mechanisms were introduced (e.g. **Bahdanau et al.** 2015) in RNN sequence-to-sequence models, allowing the decoder to *focus on* relevant parts of the input. In **"Attention Is All You Need" (Vaswani et al. 2017)**, the authors abandoned recurrence entirely and built an architecture solely from *self-attention* and feed-forward layers. In a transformer encoder, each token **attends to all others** in the sequence, enabling parallel processing and capturing global context. This innovation underlies modern LLMs. Transformer-like models (GPT, BERT, etc.) quickly surpassed RNNs on benchmarks such as GLUE, translation, and question answering.

---

## 2. Transformer architecture

A **Transformer block** consists of:

1. A multi-head self-attention layer
2. Add-and-norm
3. A feed-forward network (FFN)
4. Another add-and-norm

In self-attention, each input token is projected to query $Q$, key $K$, and value $V$ vectors. The attention output is a weighted sum of values:

$$
\text{Attention}(Q,K,V) = \text{softmax}\!\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

The scaling by $\sqrt{d_k}$ keeps gradients stable. **Multi-head attention** runs several attention "heads" in parallel — each head uses different learned linear projections $W_i^Q, W_i^K, W_i^V$, computes attention independently, and the head outputs are concatenated and projected by $W^O$:

$$
\text{MultiHead}(Q,K,V) = \text{Concat}(\text{head}_1, \dots, \text{head}_h)\,W^O
$$

$$
\text{head}_i = \text{Attention}(QW_i^Q,\; KW_i^K,\; VW_i^V)
$$

Multi-head attention allows the model to jointly attend to information from different subspaces and positions. After attention, a **residual connection** and layer normalization produce an intermediate representation, followed by a position-wise feed-forward network (two linear layers with ReLU) and another add-and-norm.

To encode token order without recurrence, transformers add **positional encodings** to input embeddings (e.g. sine/cosine functions or learned vectors).

```
Input Embeddings + Positional Encoding
        ↓
Multi-Head Self-Attention
        ↓
Add & LayerNorm
        ↓
Feed-Forward (2-layer MLP)
        ↓
Add & LayerNorm
        ↓
Block Output
```

*Transformer encoder block. Tokens are processed by multi-head self-attention (with queries, keys, values), followed by residual add-and-norm, a feed-forward network, and another add-and-norm.*

Each transformer layer has **O(n²d)** compute (with $n$ = sequence length, $d$ = hidden size) due to all-pairs dot products. By contrast, an RNN layer is O(n·d²) sequentially. This parallel structure makes transformers efficient on modern hardware. In practice, stacks of ~12–96 layers are used (e.g. 12 for BERT-base, 96 for GPT-4 style models), with hundreds of millions to trillions of parameters.

---

## 3. Training pipeline

### 3.1 Pretraining data and objectives

LLMs are **pretrained** on vast text corpora. For example, GPT-3 (175B parameters) used a filtered Common Crawl (60% of data, ~300B tokens), plus WebText2 (from Reddit), books, and Wikipedia. Data is deduplicated and filtered for quality. Tokenization splits text into subword tokens via **Byte-Pair Encoding (BPE)** or **WordPiece**. BPE (used by GPT-2) merges frequent byte sequences into tokens, while WordPiece (used by BERT) similarly builds subwords with a prefix (e.g. `##`). Both methods produce a vocabulary of typically 30–50k tokens to balance granularity and coverage.

Common pretraining objectives are:

- **Autoregressive Language Modeling (next-token prediction):** The model predicts each token given all previous ones. Used by the GPT series, Bloom, etc. The loss is cross-entropy of predicting the next token, using causal (masked) self-attention.
- **Masked Language Modeling (MLM):** The model masks a random subset of tokens (e.g. 15%) and predicts them from both left and right context. Used by BERT, RoBERTa, and DistilBERT. The MLM objective (a "cloze" task) enables bidirectional context.
- **Seq2Seq / Encoder-Decoder:** Models like T5/BART treat tasks as text-to-text. They may use a prefix token indicating the task and train to output target sequences via an autoregressive decoder.

| **Pretraining Objective**      | **Task**                                          | **Example Models**           |
|-------------------------------|---------------------------------------------------|------------------------------|
| Autoregressive LM (next-token) | Predict $w_t$ given $w_{<t}$ (causal)            | GPT, GPT-2, GPT-3/4, Bloom  |
| Masked LM (MLM)               | Mask random tokens, predict them                  | BERT, RoBERTa, DistilBERT   |
| Prefix/Seq2Seq LM             | Generate target from input prefix                 | T5, BART, UL2               |

*Table 1: Common pretraining objectives and example models.*

### 3.2 Compute and scaling laws

Training these models demands enormous computation. GPT-3 required on the order of $3 \times 10^{23}$ floating-point operations — roughly hundreds of GPU-years. Large models obey **scaling laws**: Kaplan *et al.* (2020) showed that validation loss (cross-entropy) follows power-laws in model size $N$, dataset size $D$, and compute $C$. Specifically, for transformer LMs at optimal data/model scaling, loss $L$ vs compute $C$ follows roughly $L \propto C^{-0.076}$ over many orders of magnitude. This means *every 10× increase in compute yields a fixed fractional drop in loss*.

The scaling laws imply **sample-efficiency**: larger models achieve lower loss with fewer data/updates. They also predict an optimal allocation of a fixed compute budget among $N$, $D$, and training steps. Crucially, performance depends most on *scale* (parameters, data, compute) and much less on architectural details like width vs depth. In practice, this drives ever-larger models trained on internet-scale data.

### 3.3 Tokenization and data pipeline

Before training, raw text is cleaned and tokenized. Common pipelines use libraries like Hugging Face **Tokenizers**. For example:

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("gpt2")  # byte-level BPE tokenizer
tokens = tokenizer("Hello world!", return_tensors="pt")
print(tokens.input_ids)  # tensor of token IDs
```

For BERT, WordPiece is used: `"unbelievable"` → `["un", "##believable"]`. The pipeline then creates batches of tokenized sequences (fixed max length, with padding) for training.

---

## 4. Fine-tuning and instruction alignment

After pretraining, LLMs are **fine-tuned** for tasks or to align with desired behaviors. Fine-tuning can be supervised (task-specific data) or rely on reinforcement learning from human feedback (RLHF).

In RLHF (used in InstructGPT and ChatGPT), the process involves:

1. Collect *demonstrations*: human-written answers for many prompts.
2. Fine-tune the pretrained LLM (supervised) on this dataset — **Supervised Fine-Tuning (SFT)**.
3. Collect *comparisons*: humans rank multiple model outputs per prompt.
4. Train a *reward model* on these rankings.
5. Use policy optimization (PPO) to tune the LLM to maximize the reward.

Ouyang *et al.* (2022) showed that InstructGPT (aligned GPT-3) was preferred by humans and was more truthful and less toxic than raw GPT-3. Notably, InstructGPT's 1.3B model outperformed the 175B GPT-3 on human preference despite having 100× fewer parameters — a striking demonstration of RLHF's impact.

This yields **instruction-tuned** models that follow user prompts more reliably. Recent LLMs (GPT-4, Claude, etc.) rely on sophisticated variations of RLHF or "constitutional" AI to ensure safer outputs.

---

## 5. Inference and decoding

At inference time, LLMs produce tokens sequentially using the learned distribution $P(w_t \mid w_{<t})$. Various **decoding strategies** trade off quality and diversity:

- **Greedy search:** Pick the highest-probability next token at each step. Simple but prone to repetitive or suboptimal output.
- **Beam search:** Keep the top $k$ sequences (beams) at each step, expanding all and selecting the best $k$. Explores more possibilities, often used in translation, but can still lack diversity.
- **Sampling:** Randomly sample from the token distribution (`do_sample=True`). Generates more varied output.
- **Top-*k* sampling:** Restrict sampling to the $k$ most probable tokens at each step (zero out the rest).
- **Top-*p* (nucleus) sampling:** Restrict to the smallest set of tokens whose cumulative probability ≥ $p$ (e.g. 0.9). Adapts the cutoff dynamically.
- **Temperature:** Scale logits by $1/T$ before softmax. Higher temperature ($T > 1$) flattens the distribution (more randomness); lower $T$ sharpens it.

Hugging Face Transformers makes decoding straightforward:

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
sample_ids = model.generate(
    **inputs,
    max_new_tokens=50,
    do_sample=True,
    top_k=50,
    top_p=0.95,
    temperature=0.8
)
print(tokenizer.decode(sample_ids[0]))
```

| **Decoding Method**        | **Description**                                                              |
|---------------------------|------------------------------------------------------------------------------|
| Greedy                    | Choose highest-probability token each step. Quick but can repeat.           |
| Beam search               | Keep $B$ best sequences (beams) each step. Searches more but is slower.     |
| Top-*k* sampling          | Randomly sample from the top $k$ tokens (multinomial).                       |
| Top-*p* (nucleus) sampling| Sample from the smallest set of tokens with cumulative probability ≥ $p$.   |
| Temperature scaling       | Divide logits by $T$ before softmax (higher $T$ → more randomness).         |

*Table 2: Common decoding strategies.*

---

## 6. Evaluation metrics

LLM performance is evaluated both intrinsically and on downstream tasks.

- **Perplexity (PPL):** A standard intrinsic metric, defined as $2^{\text{cross-entropy}}$. Lower perplexity means better predictive performance. It measures how well the model predicts held-out text — the "weighted average branching factor" of the language model.
- **Downstream benchmarks:** Models are fine-tuned or prompted on tasks (QA, classification, generation) and scored by task-specific metrics. Examples: GLUE/SuperGLUE for language understanding (accuracy/F1), SQuAD for QA (F1/EM), BLEU/ROUGE for translation or summarization, accuracy on common sense tests, etc.
- **Bias/toxicity metrics:** Specialized tests (e.g. RealToxicityPrompts) measure generation of harmful content. Truthfulness benchmarks (TruthfulQA) evaluate factual accuracy. Hallucination can be measured by comparing generated facts to ground truth (often manually).

Improvement in perplexity generally correlates with downstream gains, but not perfectly — better modeling is a necessary but not sufficient condition for task performance. Thus LLMs are evaluated on many tasks and held-out datasets to ensure general competence.

---

## 7. Common failure modes and biases

Despite their power, LLMs have well-known limitations:

- **Hallucinations / factual errors:** They can produce plausible but incorrect or invented information, since they only model patterns in training text and have no grounded truth-check. RLHF and retrieval-augmented generation (RAG) are used to mitigate this.
- **Bias and stereotypes:** Models trained on internet data reflect its biases. Studies show models can generate stereotypical or prejudiced content (gender, race, religion). These biases are "internet-scale" — whatever is common in the data is amplified.
- **Toxicity/offensiveness:** Unfiltered text corpora contain toxic or offensive language, which LLMs may reproduce, especially on edge-case prompts.
- **Context length limits:** Transformers have a maximum context length (e.g. 2048 tokens, often 8k–32k in modern models). Long documents get truncated or require expensive mechanisms such as long-range attention or memory augmentation.
- **Ambiguity and instruction failure:** Without careful prompting, models may misinterpret instructions or produce unhelpful answers. Even instruction-tuned models can break down on adversarial or ambiguous prompts.
- **Adversarial prompts:** Cleverly crafted inputs ("jailbreaks") can bypass safety filters, causing models to output disallowed content or behave undesirably.

These failure modes arise from training data gaps and model design. Hallucination partly reflects that maximum likelihood training (or RLHF) does not penalize ungrounded but coherent statements. Bias comes from skewed corpora.

---

## 8. Safety and mitigation strategies

To reduce risks, practitioners employ multiple strategies:

- **Data curation/filtering:** Remove or downweight toxic content, personal data, or copyrighted text from pretraining corpora. GPT-3, for instance, filtered Common Crawl down to high-quality text.
- **Reinforcement learning from human feedback (RLHF):** Aligns models with human values and reduces undesirable outputs. InstructGPT demonstrated lower toxicity and more truthful answers while maintaining fluency.
- **Rule-based or AI-based filters:** Systems like OpenAI's moderation API or "constitutional" AI frameworks post-process outputs to flag or edit harmful content. Recent work even uses large language models as feedback to shape safe behavior.
- **Adversarial testing (red-teaming):** Teams intentionally probe models for harmful behavior; findings inform additional fine-tuning or prompt engineering.
- **Ongoing alignment research:** Including "safe completion" techniques, continual fine-tuning on new safety data, and transparency measures.

No mitigation is perfect, and the field treats alignment as ongoing work. Combining content filtering, RLHF, and careful monitoring is the current best practice for reducing harmful outputs.

---

## 9. Practical usage

Here is basic usage with Hugging Face Transformers (PyTorch). First, a forward pass through a pretrained causal LM (GPT-2):

```python
from transformers import AutoTokenizer, AutoModelForCausalLM
import torch

tokenizer = AutoTokenizer.from_pretrained("gpt2")
model = AutoModelForCausalLM.from_pretrained("gpt2")

text = "Artificial intelligence is"
inputs = tokenizer(text, return_tensors="pt")
outputs = model(**inputs)      # forward pass
logits = outputs.logits        # shape [1, seq_len, vocab_size]

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

This uses nucleus sampling with temperature. The `generate()` API supports all strategies mentioned above.

---

## 10. Complexity and compute estimates

Transformer layers scale as $O(n^2 d)$ per layer, where $n$ is sequence length and $d$ is hidden size. This limits context length due to memory and time. Training costs grow roughly as $O(N \cdot S \cdot d^2)$, where $N$ is the number of parameters and $S$ is the number of steps.

For example, training GPT-3 (~175B parameters) took on the order of $3 \times 10^{23}$ floating-point operations. By scaling laws, even larger models require exponentially more compute (GPT-4 likely exceeded $10^{25}$ FLOPs). Inference cost per token is much lower: roughly a few hundred FLOPs per parameter. A 100B-parameter model needs on the order of $10^{13}$ FLOPs to generate 100 tokens.

Memory-wise, storing $n$ tokens' activations (for backprop) also scales as $O(nd)$ per layer. This is why sequence lengths are typically capped on many models, although techniques like sparse attention or chunking can extend this.

---

## 11. Future directions

LLMs continue to evolve rapidly across several fronts:

- **Efficiency:** Sparse and mixture-of-experts architectures (e.g. Switch Transformers) aim to reduce compute per token.
- **Long-context models:** New attention variants (Linformer, Performer) or recurrent memory augmentations aim to handle much longer contexts.
- **Multimodality:** Models like PaLM-E or GPT-4V integrate vision or other modalities with language.
- **Retrieval integration:** Retrieval-augmented LMs (e.g. RETRO, RAG) combine a frozen LLM with an external database to reduce hallucination.
- **Finer alignment:** Continued work on safer RLHF, "constitutional AI", and grounding with real-world data.

Open problems include truly safe alignment, understanding model internals, and reducing the carbon footprint of large-scale training. Given scaling trends and the level of community focus, LLMs will likely become more capable and specialized (e.g. domain-tuned), while research seeks to make them more reliable and controllable.

---

## References

- Vaswani *et al.*, "Attention Is All You Need," NeurIPS 2017.
- Devlin *et al.*, "BERT: Pre-training of Deep Bidirectional Transformers," ACL 2019.
- Brown *et al.*, "Language Models are Few-Shot Learners" (GPT-3), NeurIPS 2020.
- Kaplan *et al.*, "Scaling Laws for Neural Language Models," arXiv 2020.
- Ouyang *et al.*, "Training language models to follow instructions with human feedback" (InstructGPT), NeurIPS 2022.
- Hugging Face Transformers documentation — Decoding strategies.
- Jurafsky & Martin, *Speech and Language Processing* (3rd ed.).
- Mu *et al.*, "Rule Based Rewards for Language Model Safety," OpenAI technical report, 2023.
