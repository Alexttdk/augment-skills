---
type: agent_requested
description: Fine-tune LLMs with LoRA/QLoRA/full FT: dataset preparation, PEFT comparison, hyperparameter selection, evaluation metrics, and training workflow using HuggingFace/TRL.
---

# LLM Fine-Tuning

## When to Use
Activate when:
- Adapting an LLM to a specific domain, style, or task format
- Implementing instruction following, classification, or code generation
- Choosing between full fine-tuning, LoRA, QLoRA, or prompt engineering
- Setting up training pipelines with HuggingFace Transformers, TRL, or Axolotl
- Evaluating fine-tuned model quality (perplexity, BLEU, LLM-as-judge)

## Decision: Fine-Tune vs Alternatives

```
Achievable with prompting alone?
└─ Yes → System prompt + few-shot (cheapest, try first)
└─ No ↓
Need real-time or proprietary knowledge?
└─ Yes → RAG first (no training cost)
└─ No ↓
Need consistent style/format/domain behavior?
└─ Yes → Fine-tune
   ├─ VRAM < 8GB  → QLoRA (4-bit)
   ├─ VRAM 8–24GB → LoRA
   └─ VRAM 80GB+ / TPU → Full fine-tuning
```

## PEFT Method Comparison

| Method | VRAM (8B model) | Quality | Use Case |
|--------|----------------|---------|----------|
| Full FT | ~64 GB | ★★★★★ | Small models, maximum quality, abundant compute |
| LoRA (r=16) | ~18 GB | ★★★★☆ | Most production use cases |
| QLoRA (r=16) | ~6 GB | ★★★★☆ | Single consumer GPU (RTX 3090/4090) |
| Prefix Tuning | ~4 GB | ★★★☆☆ | Generation tasks, interpretable prompts |
| IA³ | ~3 GB | ★★★☆☆ | Inference-speed critical, extreme efficiency |

**Memory by model:** Llama 3.1 8B: 18GB/6GB (LoRA/QLoRA). Llama 3.1 70B: 160GB/48GB. Mistral 7B: 16GB/5GB.

## Dataset Preparation

### Supported Formats
```python
# Alpaca (single-turn instruction)
{"instruction": "Summarize this text:", "input": "Article...", "output": "Summary."}

# ShareGPT (multi-turn)
{"conversations": [{"from": "human", "value": "..."}, {"from": "gpt", "value": "..."}]}

# OpenAI messages (recommended for instruct models)
{"messages": [{"role": "system", "content": "..."}, {"role": "user", "content": "..."},
              {"role": "assistant", "content": "..."}]}
```

### Dataset Size Guidelines
| Task | Minimum | Recommended |
|------|---------|-------------|
| Classification | 100/class | 500+/class |
| Instruction following | 1,000 | 5,000–10,000 |
| Domain adaptation | 5,000 | 20,000+ |
| Code generation | 2,000 | 10,000+ |
| Multi-turn chat | 1,000 convos | 5,000+ |

### Quality Pipeline
```python
# 1. Quality filter (remove refusals, boilerplate)
bad_patterns = [r"I cannot", r"As an AI", r"I don't have access"]
filtered = [ex for ex in data
            if not any(re.search(p, ex["output"], re.I) for p in bad_patterns)
            and len(ex.get("output", "").strip()) > 20]

# 2. Deduplication (exact on instruction)
seen = set()
unique = [ex for ex in filtered
          if ex["instruction"] not in seen and not seen.add(ex["instruction"])]

# 3. Length filter (remove too short/long)
valid = [ex for ex in unique
         if 10 <= len(tokenizer.encode(ex["instruction"])) <= 2048]

# 4. Train/val split
from sklearn.model_selection import train_test_split
train, val = train_test_split(valid, test_size=0.1, random_state=42)
```

## LoRA / QLoRA Configuration

```python
from peft import LoraConfig, get_peft_model, TaskType

lora_config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=16,               # Rank: 4–64 (↑ complex tasks, ↓ small datasets to prevent overfit)
    lora_alpha=32,      # Scaling: typically 2× rank
    lora_dropout=0.05,  # Regularization: 0.0–0.1 (↑ for small datasets)
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj",
                    "gate_proj", "up_proj", "down_proj"],  # Llama/Mistral
    bias="none",
)
model = get_peft_model(model, lora_config)
model.print_trainable_parameters()  # e.g., 0.17% of params
```

```python
# QLoRA: 4-bit base model + LoRA adapters (~6GB for 8B models)
from transformers import BitsAndBytesConfig
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True, bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16, bnb_4bit_use_double_quant=True
)
model = AutoModelForCausalLM.from_pretrained(model_id, quantization_config=bnb_config, device_map="auto")
from peft import prepare_model_for_kbit_training
model = prepare_model_for_kbit_training(model, use_gradient_checkpointing=True)
# Then apply LoraConfig as above
```

**Target modules by architecture:**
- Llama 2/3, Mistral, Qwen2: `q_proj, k_proj, v_proj, o_proj, gate_proj, up_proj, down_proj`
- Falcon: `query_key_value, dense, dense_h_to_4h, dense_4h_to_h`
- GPT-2: `c_attn, c_proj, c_fc` | Phi: `q_proj, k_proj, v_proj, dense, fc1, fc2`

**LoRA rank guide:** Simple classification → r=4–8. Instruction following → r=8–16. Complex reasoning/code → r=32–64. Dataset < 1K → lower rank to prevent overfit.



## Training Workflow

```python
from transformers import TrainingArguments
from trl import SFTTrainer

training_args = TrainingArguments(
    output_dir="./ft-output",
    num_train_epochs=3,
    per_device_train_batch_size=4,
    gradient_accumulation_steps=4,       # Effective batch = 16
    learning_rate=2e-4,                  # LoRA: 1e-4–3e-4. Full FT: 1e-5–5e-5
    lr_scheduler_type="cosine",          # cosine recommended for most cases
    warmup_ratio=0.03,                   # 3–10% warmup; ↑ for small datasets
    logging_steps=10,
    eval_strategy="steps",
    eval_steps=100,
    save_strategy="steps",
    save_steps=100,
    save_total_limit=3,
    load_best_model_at_end=True,
    metric_for_best_model="eval_loss",
    bf16=True,
    gradient_checkpointing=True,
    gradient_checkpointing_kwargs={"use_reentrant": False},
    optim="paged_adamw_8bit",            # Memory-efficient; use adamw_torch for full FT
    max_grad_norm=0.3,                   # Lower for QLoRA/LoRA (1.0 for full FT)
    group_by_length=True,                # Group similar-length seqs → less padding waste
    report_to="wandb"
)

trainer = SFTTrainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
    eval_dataset=eval_dataset,
    tokenizer=tokenizer,
    max_seq_length=2048,
    packing=True,                        # Pack short sequences → better GPU utilization
    dataset_text_field="text"            # or use formatting_func= for custom templates
)
trainer.train()
```

**Merge adapter after training:**
```python
from peft import PeftModel
base = AutoModelForCausalLM.from_pretrained(base_model_id, torch_dtype=torch.bfloat16)
model = PeftModel.from_pretrained(base, "./ft-output/checkpoint-best")
merged = model.merge_and_unload()
merged.save_pretrained("./merged-model")
tokenizer.save_pretrained("./merged-model")
```

## Hyperparameter Quick-Ref

| Scenario | LR | Epochs | Per-device Batch | Grad Accum | Weight Decay |
|----------|-----|--------|-----------------|------------|--------------|
| QLoRA, < 1K examples | 1e-4 | 5 | 2 | 8 | 0.05 |
| LoRA, 1K–10K examples | 2e-4 | 3 | 4 | 4 | 0.01 |
| Full FT, > 10K examples | 2e-5 | 2 | 8 | 2 | 0.01 |

**Scheduler:** `cosine` (default). `constant_with_warmup` for very short runs. `cosine_with_restarts` for long training.

## Evaluation Strategy

### Automated Metrics
| Metric | Good For | Target |
|--------|----------|--------|
| Perplexity (↓) | Language modeling fit | Lower than base model |
| BLEU | Translation, structured output | Task-dependent baseline |
| ROUGE-L | Summarization | > 0.50 |
| BERTScore F1 | Semantic similarity | > 0.85 |
| Accuracy / F1 | Classification | > 0.80 |

### LLM-as-Judge (for open-ended quality)
```python
judge_prompt = """Rate this response 1–5 on helpfulness, accuracy, coherence.
Input: {input}
Reference answer: {reference}
Model response: {prediction}
Return JSON: {{"helpfulness": N, "accuracy": N, "coherence": N}}"""

result = json.loads(gpt4o.invoke(judge_prompt.format(...)))
```

### Standard Benchmarks
```bash
lm_eval --model hf --model_args pretrained=./merged-model \
        --tasks hellaswag,arc_easy,arc_challenge,mmlu,gsm8k \
        --num_fewshot 0 --output_path ./eval_results
```

## Common Pitfalls

| Problem | Symptom | Solution |
|---------|---------|----------|
| Loss not decreasing | Flat training curve | Try 10× LR; verify `requires_grad=True` on adapter params |
| Loss spikes | Sudden jumps | Reduce LR; add warmup; lower `max_grad_norm` |
| Overfitting | Eval loss diverges from train | Reduce epochs; increase `lora_dropout`; reduce rank |
| OOM errors | CUDA out of memory | Use QLoRA; reduce batch; enable gradient checkpointing |
| Catastrophic forgetting | Base capabilities degraded | Lower LR; fewer epochs; target only attention layers |
| Adapter not loading | Dimension mismatch error | Verify `target_modules` matches base model architecture |

## References
- Source: https://github.com/Jeffallan/claude-skills/tree/main/fine-tuning-expert
- HuggingFace PEFT: https://huggingface.co/docs/peft
- TRL SFTTrainer: https://huggingface.co/docs/trl/sft_trainer
- QLoRA paper: https://arxiv.org/abs/2305.14314
