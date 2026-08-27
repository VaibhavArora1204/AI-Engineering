# SLM Practical Knowledge — Ops & Infra Reference

---

## 1. What "Small" Means for Deployment Decisions

### What it is operationally

- "Small language model" in 2024–2025 practice means roughly **0.5B to ~13B parameters**. The industry has no hard line; what matters is the deployment profile, not the label.
- **Parameter count** is the number you see in the model name (e.g., Phi-3-mini-4k is 3.8B params, Qwen2.5-7B is 7.6B params, Gemma-2-2B is 2.6B). This is the total weight count.
- **Active parameters** differ from total params when the model uses **Mixture-of-Experts (MoE)**. Example: Qwen2.5-MoE has 14.3B total params but only ~2.7B are active per token ([Qwen2.5 Technical Report](https://arxiv.org/abs/2412.15115)). You pay memory for the full 14.3B at load time but compute cost per token is closer to a 3B dense model. This is why MoE models appear "cheap to run" but still need big VRAM.
- **Memory bandwidth footprint** is what actually determines your latency and hosting cost. During autoregressive generation (token-by-token), the bottleneck is reading all model weights from GPU memory once per token. This makes the model **memory-bandwidth bound**, not compute bound, for most SLM serving at low batch sizes.

### Why this breaks

- **Marketing-number confusion**: A team provisions a GPU for a "3B model" but the model has a 128K context. At 4K context with 1 user, KV cache is small. At 32K context with 8 concurrent users, the **KV cache alone** can exceed the model weight memory. You OOM not because the model is too big, but because nobody budgeted for KV cache memory.
- **MoE memory trap**: You see "active params: 2.7B" and think it fits on a 4GB edge device. It doesn't — you still need to load all 14.3B params. The "active params" number only helps you estimate *compute* cost (and therefore latency per token), not memory cost.
- **Throughput misestimation**: Two 7B models with different architectures (e.g., different number of attention heads, different intermediate sizes) can have measurably different tokens/sec on the same GPU because of how they hit memory bandwidth. You cannot compare models by param count alone.

### What to check/monitor

- **Actual GPU memory usage** at your expected concurrency and context length — not just model weights. Formula estimate: `total_memory ≈ model_weights + KV_cache_per_user × max_concurrent_users + framework_overhead`. For KV cache per token per layer: `2 × num_layers × num_kv_heads × head_dim × precision_bytes`.
- For MoE models: check **total** param count for memory planning, **active** param count for latency estimation.
- Run a load test with realistic prompt + generation lengths at target concurrency before committing to a GPU SKU. Don't trust napkin math from param count.
- **Key metric**: bytes-per-token-generated. This is `model_size_bytes / tokens_per_second`. If this is close to your GPU's memory bandwidth spec, you're bandwidth-saturated (normal for SLMs at low batch). If it's much lower, you have overhead problems.

---

## 2. Serving Realities

### What it is operationally

You have a model. You need to get it answering requests at target latency and throughput. Here's where the serving stack choices actually matter.

### Batching behavior

- **Continuous batching** (also called "inflight batching" or "iteration-level batching") is the standard now. Instead of waiting for a full batch of requests to arrive, the server processes tokens for multiple requests simultaneously, adding new requests mid-generation. This is what vLLM, TGI, and TensorRT-LLM all implement.
- Without continuous batching (naive static batching), a request that generates 10 tokens waits for a request generating 500 tokens to finish. This destroys tail latency.
- SLMs are small enough that at low batch sizes (1–4), you're heavily **memory-bandwidth bound** — the GPU spends most of its time reading weights, not computing. Increasing batch size up to a point is essentially "free" in latency because the extra compute is hidden behind the memory read time. This is called the **"free lunch" region** of batching.
- **Throughput plateau**: Beyond a certain batch size, you become compute bound or KV-cache-memory bound. Throughput flattens. For a 7B model on an A10G (24GB), this often hits around batch 16–32 with 2K context, earlier with longer contexts.

### Cold start vs warm latency

- **Cold start** = loading model weights from disk into GPU memory. For a 7B model in FP16, that's ~14GB of weights. On cloud instances with network-attached storage, this can take 30–120 seconds. On local NVMe, 5–15 seconds.
- **Warm latency** = time-to-first-token (TTFT) and time-per-output-token (TPOT) once the model is loaded. For a 3B model on a T4 GPU, expect TTFT of 50–200ms (depending on prompt length) and TPOT of 15–40ms.
- SLM advantage: cold start is much faster than large models, making them viable for scale-to-zero / serverless patterns. A 3B model can cold start in <10s on a good instance.
- **Trap**: if you're using serverless GPU infra (e.g., Modal, Runpod Serverless, Lambda), model loading happens on every cold start. Pre-baking the model into the container image or using a shared filesystem cache is mandatory for acceptable cold start.

### Quantized vs full-precision serving trade-offs

| Format | Size (7B model) | Latency impact | Quality impact | When to use |
|--------|-----------------|----------------|----------------|-------------|
| FP16/BF16 | ~14 GB | Baseline | Baseline | When you have GPU headroom and need maximum quality |
| INT8 (W8A8) | ~7 GB | ~1.5–2× faster TPOT | Minimal for most tasks | Default for production GPU serving |
| INT4 (W4A16) | ~3.5 GB | ~2–3× faster TPOT on GPU, huge win on CPU | Noticeable on arithmetic, complex reasoning | Edge/CPU deployment or when GPU memory is tight |
| GGUF Q4_K_M | ~4 GB | Optimized for CPU | Comparable to naive INT4, sometimes better | CPU-only or hybrid CPU+GPU (llama.cpp) |

### Serving stack decision tree

**vLLM** ([docs](https://docs.vllm.ai))
- Best for: GPU serving at medium-to-high throughput, OpenAI-compatible API needed
- Strengths: PagedAttention (efficient KV cache memory), continuous batching, broad model support, OpenAI API server built-in
- Weaknesses: GPU only, heavier framework, not ideal for single-request latency optimization
- Use when: you're deploying on cloud GPUs, need to handle concurrent users, want drop-in OpenAI API replacement

**TGI (Text Generation Inference)** ([HuggingFace](https://github.com/huggingface/text-generation-inference))
- Best for: HuggingFace ecosystem integration, production-grade with built-in metrics/health checks
- Strengths: gRPC + REST, token streaming, watermarking, production-ready out of the box
- Weaknesses: model support can lag behind vLLM for newer architectures
- Use when: you're already in HuggingFace ecosystem, need production features with less config

**llama.cpp / llama-cpp-python** ([GitHub](https://github.com/ggerganov/llama.cpp))
- Best for: CPU serving, edge deployment, macOS/Apple Silicon, GGUF quantized models
- Strengths: Runs anywhere (CPU, Apple Metal, CUDA, Vulkan), very low resource overhead, GGUF quantization ecosystem
- Weaknesses: Lower throughput than vLLM at high concurrency on GPU, API is less standardized (though OpenAI-compatible server exists)
- Use when: you need CPU-only deployment, on-device/edge, running on consumer hardware, or Apple Silicon

**ONNX Runtime** ([Microsoft](https://onnxruntime.ai))
- Best for: Cross-platform inference, mobile/edge, Windows deployment, models exported to ONNX
- Strengths: Hardware-agnostic (CPU, CUDA, DirectML, TensorRT, CoreML), optimized for Phi models specifically (Microsoft's own), good for integration into existing C#/C++ services
- Weaknesses: Model conversion step required, not all HuggingFace models export cleanly, limited continuous batching support
- Use when: deploying Phi-family models, need Windows/DirectML support, integrating into non-Python services, mobile deployment

**TensorRT-LLM** ([NVIDIA](https://github.com/NVIDIA/TensorRT-LLM))
- Best for: Maximum throughput on NVIDIA GPUs
- Strengths: Kernel-level optimization, best raw performance on NVIDIA hardware
- Weaknesses: Complex build process, NVIDIA-only, model support requires explicit engine building
- Use when: you have NVIDIA GPUs, need absolute best throughput, and can afford the engineering overhead

### Why this breaks

- **Picking the wrong stack for your hardware**: vLLM on a CPU machine = won't work. llama.cpp on an A100 with 50 QPS = leaving performance on the table.
- **Ignoring KV cache memory**: You deploy a 3B model quantized to INT4 (~2GB weights) on a 16GB GPU and think "plenty of room for large batches." At 8K context with 20 concurrent users, the KV cache blows past remaining memory. vLLM will start preempting (swapping KV cache to CPU), causing latency spikes.
- **Quantization format mismatch**: Loading a GPTQ model into a framework expecting AWQ, or using a GGUF model in vLLM (which doesn't support GGUF natively). This either crashes or silently falls back to slow paths.

### What to check/monitor

- **TTFT (time to first token)** — captures prompt processing speed. Alert if P95 exceeds your SLA.
- **TPOT (time per output token)** — captures generation speed. Determines user-perceived streaming speed.
- **Queue depth / waiting requests** — if this grows, you need more replicas or a bigger GPU.
- **KV cache utilization %** — vLLM and TGI expose this. If >90%, you're about to get preemptions and latency spikes.
- **GPU memory utilization** — track over time with concurrent users. If it's sawtoothing, you have a KV cache fragmentation or memory leak.
- Run a **load test** with production-representative prompt lengths and generation lengths before deploying. Tools: `locust`, `vegeta`, or vLLM's built-in benchmark script.

---

## 3. Quantization as an Ops Concern

### What it is operationally

Quantization reduces the precision of model weights (and sometimes activations) from FP16 (16-bit) to INT8 (8-bit) or INT4 (4-bit). The model gets smaller and faster. The question is: what quality do you lose, and how would you know?

### Common quantization methods you'll encounter

- **GPTQ**: Post-training weight quantization. Calibration-set dependent. Produces a new model checkpoint. Widely supported.
- **AWQ (Activation-Aware Weight Quantization)**: Similar to GPTQ but preserves "salient" weights at higher precision. Often slightly better quality than GPTQ at INT4. Widely supported in vLLM.
- **GGUF quantization levels** (llama.cpp ecosystem): Q4_0, Q4_K_M, Q5_K_M, Q8_0, etc. The `K` variants use a more sophisticated quantization scheme with better quality. Q4_K_M is the "sweet spot" for most people — good quality/size trade-off.
- **bitsandbytes (BnB)**: Used mainly for fine-tuning (QLoRA), not optimized for serving speed. Don't use for inference in production.
- **SmoothQuant / W8A8**: INT8 weight and activation quantization. Minimal quality loss for most tasks. Default recommendation for GPU serving.

### What breaks first when you quantize

This is the critical ops knowledge. Quality degradation from quantization is **not uniform across tasks**. It hits specific capabilities first:

1. **Arithmetic and counting** — INT4 models frequently fail at multi-digit arithmetic that FP16 handles. A 7B model that can do 3-digit addition in FP16 will start failing at 2-digit in INT4. If your pipeline relies on the model doing any math (price extraction, date arithmetic, counting items), test this specifically.

2. **Long-context coherence** — Quantization amplifies the existing weakness of small models with long contexts. A 7B model at FP16 might handle 8K context reasonably. The INT4 version might functionally degrade at 4K — it's still generating fluent text, but ignoring relevant context earlier in the window. This is **silent**: the output looks fine, it's just wrong.

3. **Instruction-following precision** — Complex system prompts with multiple constraints (output format, persona rules, guardrails) get followed less reliably. The model drops one of five rules instead of zero. This shows up as increased format violations in logs.

4. **Minority languages and code** — If you're working with non-English text or code generation, quantization degrades these disproportionately because they're underrepresented in training data and the model has less "redundancy" to absorb quantization error.

### Calibration-set mismatch — the silent killer

- GPTQ and AWQ require a **calibration dataset** during quantization. This dataset tells the quantizer which weight ranges are important to preserve.
- Default calibration sets are usually C4, WikiText, or a random subset of the training data.
- **The failure**: If your production use case is domain-specific (medical, legal, financial, code), the default calibration set doesn't represent your data distribution. The quantizer optimizes for the wrong thing. You get a model that benchmarks fine on general tasks but silently degrades on your domain.
- **How to catch it**: Quantize with a calibration set drawn from your actual production prompts (even 128–512 examples is enough). Compare the default-calibrated quant vs your domain-calibrated quant on your own eval set. If there's a gap >2% on your task metric, use the domain-calibrated version.
- Most teams skip this. It matters most for INT4; INT8 is usually robust enough that calibration set choice doesn't materially affect output.

### How to catch a bad quantized checkpoint before it ships

Do **not** use MMLU or general benchmarks. They test broad knowledge, not the specific capabilities quantization degrades.

Instead, build a small **quantization regression suite** (50–200 examples) covering:
- **Format compliance**: Give the model 10 prompts that require JSON output, markdown tables, or specific output structure. Count violations. Compare quant vs FP16.
- **Arithmetic**: 20 problems with 2-3 digit addition, subtraction, and simple multiplication. Count correct answers.
- **Instruction adherence**: 10 prompts with 3+ constraints (e.g., "respond in exactly 3 bullet points, in French, mentioning X"). Count how many constraints are met.
- **Domain-specific extraction**: 20 examples from your actual production data. Compare extraction accuracy quant vs FP16.
- **Long-context needle**: 10 examples where the answer is in the first, middle, and last third of a 4K+ context. Check if accuracy drops in the middle for the quant.

**Acceptance criteria**: If the quantized model drops more than 5% on any of these categories vs FP16, investigate. If it drops more than 10%, don't ship that quantization level.

---

## 4. Context Length Behavior

### What it is operationally

SLMs are often advertised with context windows of 8K, 32K, 128K, or even 1M tokens. The advertised number is the **maximum** the model architecture supports. The **effective** context — where the model actually uses the information reliably — is almost always shorter.

### Where the needle-in-haystack cliff sits

- For most SLMs in the 1B–7B range, effective context quality degrades significantly past 4K–8K tokens, regardless of the advertised window.
- Phi-3-mini (3.8B) claims 128K context, but independent testing shows significant degradation on retrieval tasks past 8K ([Phi-3 Technical Report, Microsoft, 2024](https://arxiv.org/abs/2404.14219) — they show strong results on their own benchmarks, but community needle-in-haystack tests tell a different story at 32K+).
- Qwen2.5-7B shows strong results up to 32K on their reported benchmarks but is known to degrade on multi-fact retrieval tasks in the second half of long contexts ([Qwen2.5 Technical Report](https://arxiv.org/abs/2412.15115)).
- Gemma-2-2B/9B are trained with 8K context. Using RoPE extension to push further is possible but reliability drops fast ([Gemma 2 Technical Report, Google DeepMind, 2024](https://arxiv.org/abs/2408.00118)).
- **Rule of thumb for planning**: For SLMs ≤7B, plan for reliable retrieval within 4K tokens. Test your specific model at your specific context length before trusting longer windows.

### How this fails in a RAG pipeline

This is the section that matters most for your RAG work.

**The "lost in the middle" problem** is well-documented ([Liu et al., 2023, "Lost in the Middle"](https://arxiv.org/abs/2307.03172)): LLMs, especially smaller ones, attend strongly to information at the **beginning** and **end** of the context, and poorly to information in the **middle**. Implications:

- **Chunk ordering matters**: If your retrieval returns 5 chunks and the most relevant one ends up at position 3, the SLM may ignore it and either hallucinate an answer from chunks 1 and 5 or make something up entirely.
- **Silent hallucination instead of abstention**: Small models are bad at saying "I don't know." When the answer is in a chunk the model ignores, it doesn't refuse — it generates a plausible-sounding but wrong answer. This is the most dangerous failure mode because downstream consumers (users, other pipeline stages) can't distinguish it from a correct answer without external validation.
- **Overstuffing context**: A common RAG antipattern is "retrieve top-20 chunks, stuff them all in the context." For SLMs, this is almost always worse than top-3. More context = more noise = higher chance the model attends to the wrong chunk.

### What production symptoms tell you this is happening

- **Answer quality degrades as retrieval count increases** — Run an A/B test: same questions, top-3 chunks vs top-10 chunks. If top-3 is better, your model has context length / attention issues.
- **Answers cite or paraphrase the first or last chunk disproportionately** — Log which chunk the answer content comes from (use a simple overlap metric or embedding similarity between answer and each chunk). If it's bimodal (first + last), you have lost-in-the-middle.
- **Increasing hallucination rate with longer prompts** — Monitor factual accuracy (even via spot-check sampling) as average prompt length grows. If they're correlated, context quality is degrading.
- **Model generates confident answers to questions it should refuse** — Track refusal rates. If they're near zero on questions where retrieval returned low-relevance chunks, the model is hallucinating instead of abstaining.

### What to do about it

- **Limit chunks to 3–5** for SLMs ≤7B. Quality almost always beats quantity here.
- **Put the most relevant chunk first or last** — if you must stuff multiple chunks, reorder by relevance with best-match first.
- **Add an explicit instruction**: "If the provided context does not contain enough information to answer, say 'Insufficient context.'" This doesn't guarantee the model will comply (see Section 5), but it helps.
- **Use a secondary check**: After generation, verify the answer is actually grounded in the provided chunks (embedding similarity, token overlap, or a cheap classifier). This catches hallucination on context-ignore.

---

## 5. Instruction-Following Collapse

### What it is operationally

You give the model a system prompt with rules. As the number and complexity of those rules increases, the model starts dropping some of them. Not all at once — it degrades gracefully, which makes it harder to catch.

### At what point small models start dropping rules

- For SLMs ≤3B (Phi-3-mini, Gemma-2-2B, Qwen2.5-1.5B): **3–4 simultaneous constraints** is where compliance starts to get unreliable. Examples of constraints: output format (JSON), language, length limit, persona, specific inclusion/exclusion rules.
- For SLMs 7B–13B (Qwen2.5-7B, Gemma-2-9B, Llama-3.2-8B): **5–7 constraints** is where you start seeing drops, especially when constraints conflict or are ambiguous.
- **The constraint type matters**: Format constraints (JSON, markdown) are followed more reliably than semantic constraints ("don't mention competitors," "always hedge uncertain claims"). Format is structural; semantic requires reasoning about what the rule means in context.

### What the failure looks like

**Partial compliance (most common):**
- System prompt says: "Respond in JSON. Include fields: answer, confidence, sources. Confidence must be a float 0–1. Sources must be an array."
- The model returns JSON but `confidence` is a string "high" instead of a float, or `sources` is a string instead of an array. 4 out of 5 constraints met.
- This is hard to catch in spot-checks because the output "looks right" at a glance.

**Format drift over a conversation or batch:**
- The model follows the format correctly for the first 10 requests, then starts drifting — shorter responses, dropping optional fields, switching from JSON to plain text with JSON-like structure.
- In a batch processing pipeline, you see increasing parse errors over time if the model is served with any state leakage between requests (shouldn't happen with proper serving, but stateful chat endpoints can leak).

**Constraint priority inversion:**
- When constraints conflict implicitly, the model "chooses" which to follow. Example: "Keep response under 50 words" + "Include a detailed explanation" → the model either gives a short, useless answer or a detailed, long one. It doesn't tell you it can't do both.
- This shows up as **bimodal output distribution** — some responses are very short, some are very long, nothing in between.

### What to check/monitor

- **Schema validation rate**: If you expect JSON output, validate every response against a JSON schema. Track pass rate over time. Any downward trend = instruction following is degrading (could be prompt changes, model update, or context length increase).
- **Field-level compliance**: Don't just check "is it valid JSON." Check each field type, presence of required fields, value range constraints. Log which fields fail most.
- **Output length distribution**: Plot output token count over time. A widening distribution or shift in mean = format drift.
- **Constraint-violation classifier**: For semantic constraints that can't be schema-validated (e.g., "don't mention competitors"), build a simple keyword or regex check and run it on every output. Log violations.

### Mitigation

- **Fewer rules, more specific**: Instead of a 15-line system prompt, use 3–5 precise rules. Move complex logic to post-processing code.
- **Repeat critical constraints at the end of the prompt**: SLMs attend to the end of the prompt more reliably (related to the recency bias in Section 4).
- **Use structured output enforcement**: vLLM and TGI support constrained decoding (JSON schema enforcement via grammar-guided generation). This guarantees structural compliance but not semantic compliance. Use it for format; use validation for semantics.
- **Post-processing > prompt engineering**: If you need a float between 0 and 1, parse the output and clamp/reject it in code. Don't trust the model to always produce valid values.

---

## 6. Fine-Tuning Ops

### What it is operationally

You have a base or instruction-tuned SLM that's close to what you need but not quite there. Fine-tuning adapts it to your specific task/domain. At SLM scale, this is actually feasible on modest hardware.

### LoRA/QLoRA vs full fine-tune feasibility

| Method | VRAM needed (7B model) | Training speed | Quality | When to use |
|--------|----------------------|----------------|---------|-------------|
| Full fine-tune (FP16) | ~60 GB (2× model + optimizer + gradients) | Baseline | Best possible | When you have multi-GPU setup AND need maximum quality AND have >50K examples |
| Full fine-tune (BF16 + DeepSpeed ZeRO-3) | ~20 GB per GPU (across 4 GPUs) | ~0.7× baseline | Same as FP16 | Same as above but on multi-GPU commodity hardware |
| LoRA (r=16–64, FP16 base) | ~18–20 GB | ~1.3× faster than full FT | 90–95% of full FT for most tasks | Default choice. Good data, specific task, moderate hardware |
| QLoRA (4-bit base + LoRA) | ~6–10 GB | ~0.8× of LoRA (quantization overhead) | 85–93% of full FT | When GPU memory is very limited (single consumer GPU). Quality gap is real but acceptable for many tasks |

- For **SLMs ≤3B**: full fine-tune is practical on a single A10G/L4 (24GB). LoRA is still recommended because it's faster and the quality gap is negligible at this scale.
- For **SLMs 7B–13B**: LoRA/QLoRA on a single 24GB GPU, full fine-tune on multi-GPU.
- **QLoRA quality caveat**: The 4-bit base model introduces quantization noise that the LoRA adapters work on top of. For high-precision tasks (classification with tight accuracy targets, structured extraction), this noise can push you below acceptance threshold. Test before committing.

### Catastrophic forgetting

- **What it is**: After fine-tuning on your task, the model gets better at your task but loses capabilities it had before — general instruction following, safety behavior, unrelated language skills.
- **Why it's worse for SLMs**: Small models have less "capacity buffer." A 70B model can absorb new knowledge without losing much old knowledge. A 3B model is already near capacity, so new fine-tuning data displaces existing capabilities faster.
- **How it shows up**: After fine-tuning a 3B model for medical QA, it starts failing at basic JSON formatting that worked before. Or it loses its safety guardrails and starts generating harmful content for prompts it previously refused.
- **LoRA partially mitigates this** because you're only modifying a small fraction of the weights. But it doesn't eliminate it — LoRA rank > 64 on a 3B model can cause meaningful forgetting.

### What regression to test after every fine-tune

"Eval loss went down" means almost nothing for production readiness. Here's the actual test protocol:

**1. Task-specific performance (obviously)**
- Your domain eval set. Accuracy/F1/whatever your task metric is.
- Must be a held-out test set the model never saw during training.

**2. Format compliance regression**
- Take 20 prompts that require structured output (JSON, specific formats). Compare pre-fine-tune vs post-fine-tune compliance rate.
- If format compliance drops >5%, your fine-tune is too aggressive (learning rate too high, too many epochs, or LoRA rank too high).

**3. General instruction following**
- Take 20 prompts from the model's original instruction-tuning domain (general QA, summarization, translation). Compare quality.
- If these degrade >10%, you have catastrophic forgetting.

**4. Safety/refusal regression**
- Take 10 prompts that the base model correctly refuses (harmful, out-of-scope). Verify the fine-tuned model still refuses.
- This is **not optional**. If you deploy a fine-tuned model that lost safety behavior, you're liable.

**5. Boundary cases for your task**
- Edge cases, adversarial inputs, out-of-distribution prompts. Fine-tuned models often become "overconfident" — they've seen your training data distribution and will force everything into that pattern.
- Example: You fine-tune for medical QA. Give it a cooking question. If it answers with medical terminology, your model is overfitting to the domain.

### What to check/monitor

- **Track all five regression categories** as a CI/CD gate. No fine-tuned model ships without passing all five.
- **Version every fine-tune artifact**: base model hash, dataset hash, hyperparameters, LoRA config, quantization config. You need to reproduce and roll back.
- **Monitor post-deploy**: Fine-tuned models can interact with production data differently than eval data. Watch for format violations and refusal rate changes in the first 48 hours after deploy.

---

## 7. Using an SLM as Judge or Classifier

### What it is operationally

You're using the SLM not to generate user-facing text, but as a pipeline component that scores, classifies, or judges other outputs. Examples: relevance scoring for retrieved chunks, evaluating generated answer quality, routing queries to the right downstream model, content classification.

### Self-preference / self-similarity bias

- When an SLM judges output from another model (or its own output), it tends to prefer text that is stylistically similar to what it would generate. This is documented broadly across LLM-as-judge literature ([Zheng et al., 2023, "Judging LLM-as-a-Judge"](https://arxiv.org/abs/2306.05685)).
- **Operational impact**: If your SLM judge evaluates outputs from two different models (one similar style, one different), the scores are not comparable. You'll systematically prefer one model over another based on style, not quality.
- **In a RAG pipeline**: If the SLM is scoring chunk relevance and the chunks come from different source document styles (formal vs. informal), it may score based on style match rather than actual relevance.

### Position bias

- SLMs exhibit strong **position bias** in comparison tasks. If you present Option A and Option B and ask "which is better?", the model systematically favors one position (usually the first or last, depending on the model).
- Phi-3 models and Qwen2.5 models both show measurable position bias in pairwise comparison tasks.
- **How this bites you**: You build an eval pipeline that presents candidate answers in a fixed order. Your "evaluation results" are partially measuring position, not quality.

### Consistency failures

- Give the same SLM the same two items to compare, but swap their order. A reliable judge should give the same verdict. SLMs frequently don't — consistency rates (same verdict regardless of order) can be as low as 60–70% for smaller models on subjective judgments.
- This means your "85% agreement with human raters" might be partially a coincidence of presentation order.

### The deterministic sanity-check

Instead of trusting agreement percentages, run these checks:

**1. Swap test (mandatory)**
- For every pairwise comparison, run it in both orders (A,B and B,A). If the model doesn't agree with itself >85% of the time, it's not reliable enough to be a judge. Use a larger model or a different approach.

**2. Calibration check**
- Include known-quality examples in your eval set: some obviously good, some obviously bad. If the SLM can't score the obviously-bad examples low and the obviously-good examples high with >95% accuracy, the model isn't calibrated for your task.

**3. Score distribution analysis**
- Plot the score distribution. If it's clustered (e.g., everything is 7-8 on a 1-10 scale), the model isn't discriminating. This is common with SLMs — they default to "pretty good" for everything.
- **Fix**: Use binary classification (good/bad) instead of Likert scales. SLMs are much more reliable at binary than at ordinal scoring.

**4. Deterministic decoding**
- Set temperature=0 and top_k=1 for judge/classifier roles. Any randomness in scoring is noise, not signal.
- Even at temperature=0, some serving frameworks introduce non-determinism from batching or floating-point non-associativity. If consistency matters, verify by running the same input 3 times and checking for identical outputs.

### What to check/monitor

- **Swap consistency rate** — track over time. If it drops, your model or data distribution has shifted.
- **Score/class distribution** — alert if the distribution shifts (e.g., suddenly everything is "relevant" or suddenly everything is "low quality"). This usually means the input distribution changed, not the model.
- **Throughput cost**: SLM-as-judge means running the model once per item evaluated. At high volume, this can be your dominant compute cost. Calculate: `items_per_day × tokens_per_judgment × cost_per_token`. Sometimes a simple embedding similarity + threshold is 100× cheaper and just as good.
- **Don't use an SLM to judge SLM output of the same family** — Phi judging Phi output has maximum self-preference bias. Use a different model family or a non-LLM method.

---

## 8. Model Selection and Versioning in Production

### What it is operationally

You need to pick an SLM for your task and keep it working as new versions come out. Model card claims and benchmark leaderboards are starting points, not answers.

### How model card claims diverge from your domain

- Model cards report aggregate benchmarks (MMLU, HumanEval, GSM8K, etc.). These test **broad** capability. Your task is **narrow**.
- A model that scores 75% on MMLU might score 90% on your medical entity extraction task but 40% on your legal clause classification task. The aggregate number tells you nothing about either.
- **Specific example**: Phi-3-mini scores well on coding benchmarks (HumanEval) relative to its size ([Phi-3 Technical Report](https://arxiv.org/abs/2404.14219)), but this doesn't predict its performance on long-form summarization or multi-turn conversation, where it notably underperforms.
- Qwen2.5-7B shows strong multilingual benchmarks ([Qwen2.5 Technical Report](https://arxiv.org/abs/2412.15115)), but performance on low-resource languages varies dramatically — test with your actual language mix.

### Why a benchmark leaderboard rank doesn't predict your task

- Leaderboard contamination: popular benchmarks leak into training data. A high score may reflect memorization, not capability. ([Oren et al., 2023, "Proving Test Set Contamination in Black Box Language Models"](https://arxiv.org/abs/2310.17623))
- Task-format mismatch: Benchmarks test multiple-choice or short-answer. Your task is extraction, summarization, or classification with specific output format. Performance doesn't transfer.
- Distribution mismatch: Benchmarks use clean, well-formed inputs. Your production inputs have typos, OCR errors, truncated text, and weird formatting.
- **Bottom line**: Benchmark rank is a filter (don't test models that score terribly), not a selector.

### What a real "swap this model" protocol looks like

When you're evaluating a new model or model version for your live pipeline:

**Step 1: Offline eval on your data (not public benchmarks)**
- Build or maintain a **golden test set**: 200–500 examples from your production data, with ground-truth labels or reference outputs.
- Run the candidate model on this set. Compute your task-specific metrics.
- Compare against the current production model on the same set.
- **Pass criteria**: Candidate must be within X% of current model (you define X based on risk tolerance, usually 2–5%).

**Step 2: Format and constraint compliance**
- Run the candidate through your format/schema validation suite (from Section 3 and 5).
- Check that it handles your system prompt constraints at least as well as the current model.

**Step 3: Latency and throughput check**
- Serve the candidate on the same hardware. Benchmark TTFT, TPOT, and throughput at your expected concurrency.
- If the candidate is slower, calculate whether the quality improvement justifies the latency/cost increase.

**Step 4: Shadow deployment**
- Run the candidate in parallel with the production model for 24–72 hours on live traffic. Don't serve its outputs to users — just log them.
- Compare outputs: format violations, output length distribution, refusal rates, and (if feasible) quality spot-checks.
- Look for **production-specific failures** that your offline eval missed: edge-case inputs, unusual prompt lengths, adversarial user inputs.

**Step 5: Gradual rollout**
- 5% → 25% → 50% → 100% traffic over days, not minutes.
- Monitor all metrics from Section 9 at each stage.
- Automated rollback trigger if any metric degrades past threshold.

### Versioning discipline

- **Pin model versions explicitly**: Don't use `latest` tags. Pin to exact model revision (HuggingFace commit hash, or your internal model registry version).
- **Store the full model config with deployment**: model name, revision, quantization method, quantization calibration set, serving framework version, decoding parameters (temperature, top_p, max_tokens).
- **Test before update**: Even "patch" model releases (Qwen2.5-7B-v1 → v1.1) can change behavior on your task. Treat every model version change as a deployment, not a dependency bump.

---

## 9. Monitoring and Drift

### What it is operationally

Your SLM is in production, generating outputs. Over time — due to input distribution shift, model updates, serving infrastructure changes, or prompt changes — it can silently degrade. "Silent" means no error, no crash, just worse outputs.

### Metrics that actually catch degradation

**Beyond accuracy — the metrics most teams miss:**

**1. Latency variance (not just median)**
- Median latency is usually stable. P95 and P99 are where problems show up.
- Increasing P99 with stable P50 = KV cache pressure, memory contention, or thermal throttling.
- Sudden latency jumps = model was silently reloaded, serving framework config changed, or a new model version was deployed without your knowledge (if using a model registry with auto-updates — don't do this).

**2. Refusal rate change**
- Track what percentage of requests the model refuses to answer (outputs like "I cannot help with that," "I don't have enough information," etc.).
- **Refusal rate dropping** is actually dangerous — it may mean the model is becoming more "helpful" in ways that bypass safety or quality guardrails (common after fine-tuning or prompt changes).
- **Refusal rate spiking** = something changed in the input distribution (more adversarial inputs) or the model/prompt was updated.

**3. Output length drift**
- Track mean and P95 output token count over time.
- Lengthening outputs = the model is becoming more verbose, possibly generating filler or repeating itself. Check for repetition patterns.
- Shortening outputs = the model is collapsing to shorter answers, possibly losing detail. Often happens after quantization or when context gets too long.

**4. Schema/format violation rate**
- If you expect structured output (JSON, specific fields, etc.), validate every response and track the violation rate.
- This is your most sensitive canary. Format violations increase before accuracy drops because the model starts losing structural discipline before it loses content quality.
- **This should be your #1 alert trigger.** A 5% increase in format violations within 24 hours = something changed, investigate immediately.

**5. Output diversity / entropy**
- If the model starts giving very similar outputs to different inputs, it may be "mode collapsing" — falling into a repetitive pattern.
- Track unique output prefixes (first 50 tokens) as a simple diversity metric. A sudden drop = repetitive outputs.

**6. Token-level anomalies**
- Watch for unusual token patterns: excessive repetition (the same phrase 5+ times), extremely long runs of whitespace or special characters, or sudden appearance of tokens not in your expected output vocabulary.
- These indicate serving issues (corrupted KV cache, GPU errors) more often than model degradation.

### A healthy alerting setup

```
CRITICAL (page on-call):
- Format violation rate > 15% (5-minute window)
- P99 latency > 3× baseline
- Error rate > 5%
- GPU OOM events

WARNING (investigate within hours):
- Format violation rate > 5% (1-hour window)
- Output length mean shifts > 20% from baseline
- Refusal rate changes > 10% from baseline
- P95 latency > 2× baseline

INFO (review daily):
- Output length distribution shift
- Score/class distribution shift (for judge/classifier use)
- Input prompt length distribution shift (may predict downstream issues)
```

### Practical implementation

- Use your existing metrics stack (Prometheus + Grafana, Datadog, CloudWatch, etc.). The model-specific metrics above are just counters and histograms — nothing exotic.
- **Log every input-output pair** (or a sample at high volume) to a data warehouse. You will need this for debugging. Retention: 30 days minimum.
- **Baseline your metrics** during the first stable week of deployment. Use these as your comparison point for alerts.
- **Input distribution monitoring**: Track prompt length, language distribution, topic distribution (even a simple keyword classifier). If inputs change, outputs will change — and it's not the model's fault. Knowing this prevents false alarms.

---

## 10. Cost / Latency / Privacy Decision Axes

### What it is operationally

"Should we use a small local model or call a big API model?" This is a decision you'll make repeatedly. Here's the actual framework, not "it depends."

### When local/small genuinely wins

**1. Privacy-constrained data**
- If your data cannot leave your network (PII, healthcare, financial, government), a local SLM is not a nice-to-have — it's a requirement.
- This is the strongest argument for SLMs. No amount of API cost savings justifies a data breach.
- Even "private" API endpoints (Azure OpenAI, Bedrock) may not satisfy compliance requirements that mandate on-premise or specific-jurisdiction processing.

**2. Latency-critical paths with high volume**
- API call latency: 200–2000ms (variable, depending on provider load, model size, network).
- Local SLM latency: 20–100ms TTFT, 10–30ms TPOT on GPU. Predictable, no network variability.
- If you're in a synchronous pipeline where the LLM call is on the critical path and you make >1000 calls/hour, the latency predictability of local serving often matters more than the raw speed.

**3. Simple classification / routing / extraction tasks**
- Tasks where a 3B model at 95% accuracy is "good enough" and a 70B model at 98% accuracy doesn't change the business outcome.
- Entity extraction, intent classification, language detection, content tagging, PII detection — these are SLM sweet spots.
- Cost comparison: GPT-4o at $5/$15 per million input/output tokens vs a Qwen2.5-3B on a single L4 GPU ($0.70/hr) serving ~500 req/sec of classification. At 1M requests/day with 200-token prompts and 20-token outputs, API cost ≈ $100–150/day, local ≈ $17/day. Local wins clearly.

**4. Offline / batch processing**
- Nightly batch processing of 100K documents for classification/extraction/summarization.
- Spin up spot instances, run the batch, tear down. No API rate limits, no throttling, predictable cost.

### When local/small is a false economy

**1. The total cost trap**
- GPU cost is not your only cost. Add up:
  - **Engineering time**: Setting up serving infrastructure, handling OOMs, tuning quantization, building eval pipelines. At $150K/yr engineer salary, 2 weeks of setup = $5,770.
  - **Ongoing maintenance**: Model updates, eval reruns, monitoring infrastructure, on-call for model-specific issues.
  - **Quality gap cost**: If the SLM is 90% accurate and the API model is 98%, that 8% gap is X support tickets, Y incorrect downstream decisions, Z user churn. Quantify this.
- **Break-even formula**: `local_cost = GPU_cost + (engineering_hours × hourly_rate) + quality_gap_cost`. Compare to `API_cost = tokens × price_per_token + (minimal_integration_time × hourly_rate)`.
- For low volume (<100 requests/day), the API almost always wins because the fixed cost of local infra isn't amortized.

**2. Tasks requiring strong reasoning**
- Multi-step reasoning, complex mathematical word problems, nuanced legal/medical analysis, long-form creative writing with specific constraints.
- SLMs can't do these reliably. Fine-tuning helps on narrow slices but doesn't give general reasoning ability.
- Sending these tasks to an SLM will generate outputs that look correct but are subtly wrong. The debugging cost of catching and fixing these is higher than the API cost.

**3. Rapidly evolving requirements**
- If your prompts, output format, or task definition changes weekly, the fine-tuning and evaluation overhead of SLMs becomes a bottleneck.
- API models handle prompt engineering changes instantly. SLM fine-tunes require training, eval, and deployment cycles.

**4. Multi-language or multi-domain requirements**
- SLMs are typically strong in 2–3 languages and weak everywhere else. If you need 10+ languages, you either need a different SLM per language cluster (ops nightmare) or a large API model.

### The actual break-even logic

```
Decision: Local SLM vs API

Inputs:
  - request_volume_per_day
  - avg_input_tokens
  - avg_output_tokens
  - required_accuracy (minimum acceptable)
  - data_privacy_requirement (boolean: must data stay local?)
  - task_complexity (simple classification / extraction / generation / reasoning)
  - latency_p95_requirement_ms
  - team_size_available_for_ml_infra

Decision:

IF data_privacy_requirement == TRUE:
    → Local SLM (non-negotiable)

IF task_complexity == "reasoning" AND required_accuracy > 90%:
    → API (SLMs won't hit this)

IF request_volume_per_day < 100:
    → API (fixed costs of local don't amortize)

IF task_complexity == "classification" OR "extraction":
    IF SLM_accuracy >= required_accuracy (test this, don't assume):
        IF request_volume_per_day > 1000:
            → Local SLM (cost wins clearly)
        ELSE:
            → API (simpler, similar cost)

IF latency_p95_requirement_ms < 200:
    → Local SLM (APIs can't guarantee this)

IF team_size_available_for_ml_infra < 1 FTE:
    → API (you can't maintain local infra part-time reliably)

DEFAULT:
    → Start with API, build eval suite, use eval suite to test SLM candidates
    → Migrate to local when volume justifies AND eval proves quality parity
```

---

## Top 3 Things to Verify Before Putting Any SLM into a Production Path

**1. Run your actual task on your actual data — not benchmarks.**
Build a 200+ example golden test set from production data. Measure your task-specific metric. If the SLM doesn't meet your accuracy threshold on this set, no amount of deployment optimization will fix it. This eliminates 50% of bad model choices.

**2. Validate output format compliance under load at your target quantization.**
Serve the quantized model at your expected concurrency and context length. Send 500+ requests. Measure schema/format violation rate. If it's >5%, either change quantization level, reduce context, or pick a different model. This catches the interaction of quantization + context length + load that no offline eval reveals.

**3. Set up monitoring for format violations and output length drift from day one.**
These are your earliest warning signals. An accuracy drop is a lagging indicator — by the time you measure it, the model has been serving bad outputs for hours or days. Format violations and length drift are leading indicators that catch degradation within minutes. If you only have time to monitor two things, monitor these two.
