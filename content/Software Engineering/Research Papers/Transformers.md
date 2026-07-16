# Transformers — How LLMs Actually Work

> **Source:** Stanford CS25 "Transformers United" (Winter 2023) — Introductory Lecture
> **Speaker:** Andrej Karpathy (ex-Tesla AI Director, OpenAI co-founder)
> **Related:** [[Spanner]] · [[Consensus Algorithms]]

> **Who this is for:** An SDE2 who is comfortable with code and systems, but has never studied machine learning or neural networks. Every concept here is explained from first principles with software engineering analogies.

---

## What Is a Transformer?

A Transformer is a type of **neural network architecture** — a mathematical function with millions of learnable parameters that takes some input and produces some output. What makes Transformers special is that this same architecture, with almost no structural changes, works extraordinarily well across:

- Natural Language Processing (GPT, BERT, ChatGPT)
- Computer Vision (ViT — Vision Transformer)
- Speech Recognition (Whisper)
- Biology (AlphaFold — protein structure prediction)
- Reinforcement Learning (Decision Transformer)
- Robotics, code generation, music, image generation

Before Transformers, each domain had its own specialized architectures, its own vocabulary, its own tricks. Transformers unified them all under one architecture. This is the central, remarkable fact about Transformers that this lecture tries to convey.

---

## Background: How We Got Here

### Before 2012 — The Feature Engineering Dark Ages

Before neural networks took over, building an AI model for, say, image classification looked like this:

1. Read 3 pages of a paper describing every possible hand-crafted feature extractor (color histograms, texture descriptors, edge detectors, geometric histograms)
2. Extract all of them from your image
3. Concatenate into a big vector
4. Feed into an SVM (Support Vector Machine) classifier
5. Get mediocre results

Every researcher had their favorite feature descriptor. Papers were unreadable across domains — NLP papers used terms like "morphological analysis," "co-reference resolution," "syntactic parsing" that computer vision people couldn't follow. Each field was a silo.

This is the engineering equivalent of solving every problem by writing specialized assembly code when a compiler exists.

### 2012 — The Neural Network Moment (AlexNet)

Researchers demonstrated that if you take a large neural network and train it on a large dataset, it learns to extract features automatically — and beats everything hand-crafted by a wide margin.

The key recipe: **scale neural networks + scale data + scale compute → better performance**

This "copy-paste recipe" spread across fields. Computer vision researchers, NLP researchers, speech researchers all started using neural networks. The vocabulary converged. Papers became cross-readable.

### 2014–2017 — RNNs and The Sequence Problem

The next challenge: how do you process **sequences** of variable length? A sentence in English has different length than its French translation. A time series has arbitrary length.

The dominant solution was **RNNs (Recurrent Neural Networks)** and **LSTMs (Long Short-Term Memory networks)**. Think of an RNN as a function that processes one element at a time, maintaining a "hidden state" (a vector) that summarizes everything it has seen so far:

```
hidden_state = 0  # initial empty memory

for word in sentence:
    hidden_state = f(hidden_state, word)  # update memory with each word

# hidden_state now "encodes" the entire sentence
```

**What worked:** RNNs were good at encoding recent history and processing sequential data.

**What didn't work:**
1. **Long-range dependencies**: RNNs struggle to connect information that is far apart. Consider: *"I grew up in France... [200 words later] ...I speak fluent ___"*. The RNN needs to remember "France" from 200 words ago to predict "French." By the time it gets there, the signal has faded.
2. **Serial computation**: you can't process word 5 until you've finished word 4. This is terrible for modern parallel GPU hardware.
3. **The encoder bottleneck**: in seq-to-seq tasks (e.g., English → French translation), the entire source sentence was compressed into a single fixed-size vector. Jamming an entire paragraph into one vector loses information.

### 2015 — Attention Mechanism (The Precursor)

A paper called "Neural Machine Translation by Jointly Learning to Align and Translate" (Bahdanau et al., 2015) introduced the key idea that became attention:

**Instead of compressing the source sentence into one vector, let the decoder "look back" at all the encoder states and decide which words to pay attention to for each step of decoding.**

Concretely: when generating the French word for "bank," the decoder looks at all English words, computes a "relevance score" for each, applies softmax to get weights that sum to 1, then takes a weighted sum of the encoder's hidden states. The word "bank" gets high weight; words like "the," "is," "a" get low weight.

This soft, differentiable lookup mechanism was called **attention**. The name "attention" was actually added by Yoshua Bengio in the final editing pass — the original name was "RNN Search."

> **Insight from Bahdanau:** He was a non-native English speaker, and during his middle-school English classes, he would look back and forth between source and target text while translating. He formalized this gaze-shift behavior as the attention mechanism. His native-language translation experience gave him the insight to design it.

The core equations:
```
score(query, key) = dot_product(query, key)       # how relevant is this key to my query?
weights = softmax(scores)                          # normalize to sum to 1
output = sum(weights[i] * values[i] for all i)    # weighted average of values
```

This is the entire mathematical heart of attention. Everything else is engineering around it.

---

## 2017 — "Attention Is All You Need"

The 2017 paper proposed something radical: **what if we delete the RNN entirely and keep only the attention mechanism?**

The paper showed that attention alone — without any recurrent processing — was sufficient to build a state-of-the-art sequence model. Moreover, without the RNN bottleneck, everything could be computed in parallel.

This architecture was called the **Transformer**.

The paper also combined several ideas simultaneously in a way that proved remarkably stable:
- Self-attention (attention where a sequence attends to itself)
- Multi-head attention (running attention multiple times in parallel)
- Residual connections (from ResNets — skip connections that stabilize training)
- Layer normalization
- Position-wise MLP blocks
- Positional encodings

Critically: **almost nothing in this 2017 architecture has changed.** The same architecture — 7+ years later — is the core of GPT-4, Gemini, Claude, BERT, and every major LLM. Hundreds of researchers tried to improve it; almost nothing stuck. The 2017 paper found a remarkably good local minimum in architecture space.

---

## The Transformer Architecture — From First Principles

### The Core Intuition: Message Passing on a Graph

Think of the Transformer as a graph where:
- Each **node** is a token (a word, subword, character, or any piece of input)
- Each node holds a **vector** of numbers (its current "understanding" of itself in context)
- Nodes communicate by passing messages to each other

The Transformer alternates between two phases:

```
Layer 1: [Communication Phase] → all nodes exchange information
Layer 1: [Computation Phase]   → each node thinks independently
Layer 2: [Communication Phase] → all nodes exchange information again
Layer 2: [Computation Phase]   → each node thinks independently
...
```

Stacking many layers of this lets information propagate and be refined.

### Phase 1: Self-Attention (The Communication Phase)

Self-attention is how nodes communicate. Each node has three things it can produce:

| | Meaning | Analogy |
|---|---|---|
| **Query (Q)** | "What am I looking for?" | A search query |
| **Key (K)** | "What do I have / what am I about?" | A document's index entry |
| **Value (V)** | "What will I actually share if someone wants me?" | A document's content |

All three are produced from the same node vector via a linear transformation (a matrix multiply):
```
Q = node_vector × W_Q    # what I'm looking for
K = node_vector × W_K    # what I advertise I have
V = node_vector × W_V    # what I'll share
```

**Communication protocol:**
1. Node `i` broadcasts its query: "Here's what I'm looking for"
2. Every node `j` that can talk to `i` broadcasts its key: "Here's what I have"
3. Compute relevance scores: `score(i, j) = dot_product(Q_i, K_j)`
4. Normalize with softmax: `weights = softmax(scores / sqrt(d_k))` — the division by `sqrt(d_k)` prevents large dot products from saturating softmax
5. Gather information: `output_i = sum(weights[j] * V_j for all j)`
6. Node `i` updates itself with this gathered information

```python
# Simplified single-node attention (from the lecture's pseudocode)
Q = linear(x, W_Q)      # query: what am I looking for?
K = linear(x, W_K)      # key: what do I have?
V = linear(x, W_V)      # value: what will I share?

scores = Q @ K.T / sqrt(d_k)   # dot product = relevance score
weights = softmax(scores)        # normalize
output = weights @ V             # weighted sum of values
```

**In English:** every token gets to ask "what am I looking for?" and every other token gets to answer "here's what I have, here's how relevant I am to you." The token then synthesizes information from all others, weighted by relevance.

### The Masking (Causal Attention)

For language models that generate text left-to-right, you have a constraint: **when predicting token at position 5, you cannot look at tokens 6, 7, 8...** because they haven't been generated yet and would give away the answer.

Solution: **mask out future positions** in the attention scores by setting them to -∞ before the softmax. After softmax, -∞ becomes 0 (zero weight), so those tokens contribute nothing.

```python
# Causal masking
mask = upper_triangular_matrix(fill=float('-inf'))  # future positions = -inf
scores = scores + mask
weights = softmax(scores)  # future positions have weight ≈ 0
```

This is the "triangular" connectivity pattern in decoder models like GPT. Position 1 can only see itself. Position 2 can see positions 1-2. Position N can see all positions 1-N.

For encoder models like BERT (which process a full sentence at once, not generate token-by-token), no mask is applied — every token sees every other token.

### Phase 2: MLP (The Computation Phase)

After nodes exchange information via attention, each node independently processes its updated representation through a **feed-forward neural network** (MLP — multilayer perceptron):

```python
output = linear_2(gelu(linear_1(x)))
```

- `linear_1` expands the vector dimension by 4× (a hyperparameter from the 2017 paper that stuck)
- `gelu` is a nonlinearity (think: a smooth version of ReLU — a function that makes the network non-linear and able to learn complex functions)
- `linear_2` projects back to original dimension

The MLP is **not** about communication — it's individual computation at each node. Think of it as each node "digesting" the information it just collected and updating its beliefs.

The 4× expansion factor means: if your model dimension is 512, the MLP expands to 2048, applies nonlinearity, then projects back to 512. This creates a "thinking bottleneck" that forces compression and abstraction.

### Multi-Head Attention

Instead of running attention once, run it `h` times in parallel with **different weight matrices** (`W_Q`, `W_K`, `W_V`):

```python
heads = [attention(x, W_Q_i, W_K_i, W_V_i) for i in range(h)]
output = concat(heads) @ W_O
```

Each head learns to attend to **different aspects** of the input simultaneously. One head might learn syntactic structure (which words are subjects/objects). Another might learn semantic similarity. Another might track co-reference (what "it" refers to). You get this specialization "for free" — it emerges from training, you don't program it.

### Residual Connections and Layer Normalization

Two more key ingredients that make training stable:

**Residual connections** (skip connections): instead of replacing the node's vector, you *add* the attention output to the original:
```python
x = x + attention(LayerNorm(x))    # add, don't replace
x = x + mlp(LayerNorm(x))          # add, don't replace
```

This means gradients can flow directly backward through the `+` operation without degrading. Without residual connections, stacking many layers causes gradients to vanish (multiply many numbers < 1 together → approaches 0).

**Layer Normalization**: normalizes the vector at each position to have mean 0 and variance 1. Prevents activation magnitudes from exploding or collapsing during training.

### Positional Encoding

Attention operates on **sets** — it has no built-in notion of order. `[A, B, C]` and `[C, A, B]` would produce the same attention outputs (just rearranged).

But for language, word order matters. Solution: **add a positional signal to each token embedding** before it enters the Transformer.

```
input[i] = token_embedding[i] + positional_embedding[i]
```

The positional embedding is a vector that encodes "I am at position i." The original 2017 paper used fixed sinusoidal functions. Modern LLMs use learned or RoPE (Rotary Position Embedding) variants.

### Putting It All Together — One Transformer Block

```
Input tokens → Token Embeddings + Positional Embeddings
                        ↓
              ┌──── Transformer Block (repeated N times) ────┐
              │  x = x + MultiHeadAttention(LayerNorm(x))   │
              │  x = x + MLP(LayerNorm(x))                  │
              └──────────────────────────────────────────────┘
                        ↓
              Language Model Head (Linear → Softmax)
                        ↓
              Probability distribution over next token
```

The output at each position is a probability distribution over the entire vocabulary. Training: minimize cross-entropy loss between predicted distribution and the actual next token.

---

## Three Transformer Variants

The architecture varies by what connectivity pattern the attention uses.

### Decoder-Only (GPT, LLaMA, Claude)

- Causal (triangular) masking: each token only sees past tokens
- Trained with **language modeling**: predict the next token at every position
- Used for: text generation, chat, code completion
- Examples: GPT-2, GPT-3, GPT-4, LLaMA, Claude

```
Input:  [The, cat, sat, on]
Target: [cat, sat, on, the]
```
Every position tries to predict the next token. A single forward pass gives you N training examples from N tokens.

### Encoder-Only (BERT, RoBERTa)

- No masking: every token sees every other token
- Trained with **masked language modeling**: randomly mask 15% of tokens, predict them from context
- Used for: text classification, sentence embeddings, information retrieval
- Examples: BERT, RoBERTa, ELECTRA

```
Input:  [The, cat, [MASK], on, the, mat]
Target: [MASK] = "sat"
```

Encoder models are bidirectional — they can use both past and future context when understanding a token. Better for classification; can't autoregressively generate.

### Encoder-Decoder (T5, original Transformer, BART)

- Encoder: processes the source (e.g., English sentence) with full attention
- Decoder: generates the target (e.g., French sentence) with causal attention + cross-attention
- **Cross-attention**: decoder's queries come from the decoder tokens, but keys and values come from the encoder's final layer — this is how the decoder "reads" the encoded source

Used for: machine translation, summarization, question answering

---

## Tokenization — How Text Becomes Numbers

Transformers don't process raw text. Every input must be converted to integers first.

**Tokenization** is the process of splitting text into chunks (tokens) and mapping each to an integer ID:

```python
text = "Hello world"
tokens = ["Hello", " world"]      # subword splitting
ids    = [15496, 995]             # each maps to an integer
```

Modern LLMs use **Byte-Pair Encoding (BPE)** or **SentencePiece** — algorithms that find the most common subword units in the training corpus. Common words are single tokens ("the" → 1 token). Rare words are split into subwords ("unbelievable" → ["un", "believ", "able"] → 3 tokens).

**Context window** = the maximum number of tokens a Transformer can process at once (the block size in the code). The attention matrix has size `context_length²`, so attention is **quadratic in sequence length**. GPT-2: 1024 tokens. GPT-4: 128k tokens. This quadratic cost is one of the major open research problems.

---

## Autoregressive Generation

After training, generating text works like this:

```python
tokens = [start_token]                    # begin with start token

while not done:
    logits = model(tokens)                # forward pass
    probs = softmax(logits[-1])           # distribution over next token
    next_token = sample(probs)            # sample (or take argmax)
    tokens.append(next_token)             # append and repeat
    
    if len(tokens) > context_window:
        tokens = tokens[-context_window:] # crop to fit
```

This is **autoregressive** generation — generate one token at a time, feed it back in, generate the next. Each forward pass is O(n) tokens (though inference has KV-caching optimizations that reuse prior computation).

The model never sees beyond its context window. After `context_length` tokens, it starts forgetting the beginning of the conversation — this is why early ChatGPT "forgot" things said early in long conversations.

---

## Applications Beyond Text

The Transformer's native data type is a **set of vectors**. Anything that can be represented as a set of vectors can be fed into a Transformer. This is why it generalizes everywhere.

### Vision Transformer (ViT)

Chop an image into 16×16 pixel patches. Treat each patch as a "token." Flatten the patch into a vector. Add positional encodings. Feed into a standard Transformer encoder.

```
Image (224×224) → 196 patches (14×14 grid) → 196 tokens → Transformer
```

This is "ridiculous" (the lecturer's word) — you're throwing away all the spatial structure that CNNs were carefully designed to exploit. But it works, especially at scale.

### Speech Recognition (Whisper)

Convert audio to a Mel spectrogram (a 2D time-frequency representation). Chop into time slices. Feed into Transformer. Pretend you're doing sequence-to-sequence from audio tokens to text tokens.

### AlphaFold (Protein Folding)

Each amino acid in a protein sequence is a token. The Transformer learns which amino acids interact with each other (the 3D contact map) via attention weights, then predicts the 3D structure.

### Decision Transformer (Reinforcement Learning)

Represent a trajectory of states, actions, and rewards as a sequence. Train a Transformer to model this sequence. At inference time, condition on a desired return and let it generate the actions that would achieve that return.

### Tesla Autopilot

Multiple camera feeds + radar + map data → chop each modality into patches → concatenate all patches into one big set → Transformer processes the combined set via self-attention → predictions for driving.

The Transformer doesn't need you to explicitly specify how camera data and radar data should interact. The self-attention mechanism figures it out from data. You just throw everything in.

---

## Why Transformers Are So Effective

Three simultaneous properties make Transformers exceptional. No prior architecture combined all three.

### 1. Expressiveness (What It Can Learn)

The attention mechanism can implement very complex, data-dependent functions. Most notably, GPT-3 demonstrated **in-context learning**: give the model a few examples in the prompt, and it learns the pattern without any gradient updates.

```
Prompt:
  "cat" → "chat" (French)
  "dog" → "chien" (French)
  "house" → ???

Model output: "maison"
```

The model wasn't fine-tuned on translation. It just read the examples in context and inferred the task. This is closer to meta-learning than classical supervised learning — the model seems to be "learning" inside its activations as it reads the prompt.

Research papers have shown that Transformers appear to implement something like gradient descent *in their forward pass* — reading examples from the context is functionally similar to doing a learning update. This is an active area of research.

### 2. Optimizability (How Well It Trains)

Training a neural network means doing gradient descent — computing how much each parameter contributed to the error and adjusting accordingly. This requires gradients to flow backward from the loss to every parameter.

In RNNs, the computation graph is **long and thin** — to get a gradient from the end of a sequence back to the beginning, it has to pass through every time step, multiplying through the same weights repeatedly. Small gradients collapse to 0 (vanishing gradient). Large ones explode.

Transformers have a **short and wide** computation graph:
- Residual connections create a "highway" where gradients flow directly from loss to early layers without multiplication
- Layer normalization keeps activation magnitudes controlled throughout
- The depth (number of layers) is small relative to sequence length

Result: gradients flow cleanly, training is stable, models can be scaled to billions of parameters without special tricks.

### 3. Efficiency (GPU Utilization)

Modern GPUs are massively parallel matrix multiplication engines. A V100 GPU can do ~100 trillion floating point operations per second, but only if you can structure your computation as large matrix multiplies.

RNNs are fundamentally **serial** — you can't compute step 5 until step 4 is done. This leaves most of the GPU idle.

Transformers compute all positions **in parallel**:
```python
# All positions processed simultaneously
Q = X @ W_Q    # [batch, seq_len, d_model] × [d_model, d_k] = big matrix multiply
K = X @ W_K    # same
V = X @ W_V    # same
scores = Q @ K.T  # another big matrix multiply
output = softmax(scores) @ V  # another big matrix multiply
```

Everything is large matrix multiplies. The GPU is always busy. This is why Transformers can be scaled to hundreds of billions of parameters — the architecture is designed from first principles to saturate GPU throughput.

**The analogy:** RNNs are like a single-threaded serial program. Transformers are like a massively parallel program. On modern hardware, parallelism wins by orders of magnitude.

---

## The Transformer as a General-Purpose Computer

One of the most interesting framings from the lecture: **GPT is a general-purpose computer that runs natural language programs.**

- The program is the prompt
- The computation is the forward pass (completing the document)
- GPT has been trained on enough varied text that it can "run" many different programs specified in natural language

RNNs are also theoretically Turing-complete (they can compute anything), but this is useless in practice if:
1. They can't be trained to discover the right computation
2. They can't execute it efficiently on available hardware

Transformers satisfy both. The architecture simultaneously optimizes expressiveness, optimizability, and efficiency — and this combination, more than any single algorithmic innovation, explains why Transformers took over.

---

## Current Limitations and Open Research Problems

### 1. Quadratic Attention Cost

Attention computes a score for every pair of tokens: `O(n²)` in time and memory. A sequence of 1000 tokens requires 1,000,000 score computations. 100,000 tokens: 10 billion.

Active research areas:
- **Sparse attention**: only attend to a subset of tokens (local windows + global tokens)
- **Linear attention**: approximate attention in O(n) using kernel methods
- **State space models (Mamba)**: alternative recurrence-based architecture that avoids quadratic cost while preserving long-range modeling
- **FlashAttention**: GPU memory-efficient exact attention via tiling

### 2. No Long-Term Memory

The context window is the Transformer's only memory. Close the window, everything is gone. There is no persistent external memory between conversations.

Research directions:
- **Retrieval-augmented generation (RAG)**: store information in an external database, retrieve relevant chunks at inference time and inject into the context
- **Scratchpad mechanisms**: train the model to explicitly write working memory into the context and read from it
- **Memory-augmented Transformers**: neural memory banks that persist across contexts

### 3. Autoregressive Bottleneck

Generating text token-by-token (chunk-chunk-chunk) and committing to each token is unnatural. Humans draft and revise. A single bad early token can derail the entire output.

Research directions:
- **Speculative decoding**: run a small "draft" model to generate tokens quickly, verify with the large model in parallel
- **Diffusion language models**: generate a rough draft of the entire sequence, then iteratively refine (like image diffusion models) — allows revision, not just commitment
- **Best-of-N sampling**: generate N candidates, score them, pick the best

### 4. Controllability

Model outputs are stochastic. The same prompt can produce different outputs each run. For production systems that need deterministic behavior, this is a challenge.

Partial solutions: temperature (controls randomness), top-p/top-k sampling, constrained generation, RLHF (Reinforcement Learning from Human Feedback — how ChatGPT was trained to follow instructions).

### 5. Alignment

Making large language models reliably produce helpful, harmless, honest outputs. The raw pretraining objective (predict the next token) produces a model that will generate anything — including harmful content — because the training data contains everything.

RLHF (used in ChatGPT): collect human preference data, train a reward model on it, use RL to fine-tune the LLM to maximize the reward model's score. This is what makes ChatGPT behave like an assistant rather than a raw text predictor.

---

## The Timeline

| Year | Event |
|---|---|
| 2003 | Bengio et al. — neural language modeling with MLP (predict 4th word from 3) |
| 2012 | AlexNet — neural networks scale with data (ImageNet moment) |
| 2014 | Seq2seq — encoder-decoder RNNs for translation |
| 2015 | Bahdanau et al. — attention mechanism for seq2seq |
| 2017 | "Attention is All You Need" — the Transformer |
| 2018 | BERT — encoder-only Transformer, bidirectional |
| 2018 | GPT-1 — decoder-only Transformer |
| 2020 | GPT-3 — 175B parameters, in-context learning |
| 2021 | DALL-E (text → image), Codex (code), AlphaFold 2 (proteins) |
| 2022 | ChatGPT (RLHF fine-tuned GPT), Whisper (speech), Stable Diffusion |
| 2023 | GPT-4, LLaMA, Claude — large-scale deployment |
| 2024+ | Long-context models (100k+ tokens), multimodal (text + image + audio + video) |

---

## Key Terms Reference

| Term | Plain English Definition |
|---|---|
| **Token** | The atomic unit of input — a character, subword, or word mapped to an integer |
| **Embedding** | A token → vector mapping; each integer maps to a dense vector of numbers |
| **Attention** | A mechanism where each token computes a weighted average over all other tokens based on learned relevance |
| **Query / Key / Value** | Q: what I'm looking for; K: what I advertise; V: what I'll share |
| **Dot product** | A measure of similarity between two vectors; Q·K = relevance score |
| **Softmax** | Converts any vector of numbers into a probability distribution (sums to 1) |
| **Self-attention** | Attention where a sequence attends to itself |
| **Cross-attention** | Attention where queries come from one sequence, keys/values from another |
| **Multi-head attention** | Running attention h times in parallel with different learned weights |
| **MLP / Feed-Forward** | A small neural network applied independently to each token after attention |
| **Residual connection** | `x = x + f(x)` — adding the function output back, creating a skip path |
| **Layer normalization** | Normalizing each token's vector to mean 0, variance 1 |
| **Positional encoding** | Adding position information to token embeddings since attention is order-agnostic |
| **Causal masking** | Blocking future tokens from attending to the current position (for autoregressive models) |
| **Context window / block size** | Maximum number of tokens processed at once; attention is quadratic in this |
| **Autoregressive** | Generating one token at a time, feeding each token back in to predict the next |
| **Encoder-only** | BERT-style: full attention, no masking, used for understanding |
| **Decoder-only** | GPT-style: causal masking, used for generation |
| **Encoder-Decoder** | T5/original Transformer: encode source fully, decode target autoregressively |
| **Tokenization** | Converting text to integers (BPE, SentencePiece) |
| **BPE** | Byte-Pair Encoding — algorithm that finds common subword units |
| **In-context learning** | Learning a task from examples in the prompt without gradient updates |
| **RLHF** | Reinforcement Learning from Human Feedback — how ChatGPT was trained to follow instructions |
| **Perplexity** | A measure of how well a language model predicts a text (lower = better) |
| **Temperature** | Controls randomness of sampling; low = deterministic, high = creative/random |

---

## Sources

- Stanford CS25 "Transformers United" — Winter 2023, Introductory Lecture (Andrej Karpathy, Div Garg, Rylan Schaeffer)
- Vaswani et al., "Attention Is All You Need," NeurIPS 2017
- Bahdanau et al., "Neural Machine Translation by Jointly Learning to Align and Translate," 2015
- Brown et al., "Language Models are Few-Shot Learners" (GPT-3), NeurIPS 2020
- Dosovitskiy et al., "An Image is Worth 16x16 Words: Transformers for Image Recognition at Scale" (ViT), 2021
- Devlin et al., "BERT: Pre-training of Deep Bidirectional Transformers," 2019
- Anil et al., "Gemini," 2023
- Radford et al., "Learning Transferable Visual Models From Natural Language Supervision" (CLIP), 2021
- Karpathy, NanoGPT — minimal GPT-2 reproduction in ~300 lines of PyTorch
