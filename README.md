# ZK Constraint Auditor v0.0

A LoRA fine-tune of `Qwen2.5-Coder-1.5B-Instruct` specialized in identifying insufficient constraints in ZK proof circuits. Fine-tuned on data from the [mourningdove007/zk-constraint-data](https://github.com/mourningdove007/zk-constraint-data) GitHub repository. About 15 training examples and about 3 validation examples are far from sufficient to train a quality model. A model fine-tuned on so little data will likely behave sensibly only on very simple circuits. This version is a proof of concept.


## Intended use

- First-pass triage of Circom circuits during security audits
- Educational tool for learning constraint insufficiency patterns
- Research baseline for ZK security tooling

## Training details

| Parameter | Value |
|-----------|-------|
| Base model | Qwen2.5-Coder-1.5B-Instruct |
| Method | LoRA (MLX) |
| LoRA layers | 8 |
| Training examples | ~15 |
| Validation examples | ~3 |
| Iterations | 200 |
| Learning rate | 2e-5 |
| Framework | Circom |


## Usage (macOS)

We use [MLX](https://github.com/ml-explore/mlx-lm) for fine-tuning an existing model. Install it with:

```
pip install -U mlx-lm
```

The model can be loaded from our Hugging Face repository and used to generate a response for a user-supplied prompt.

```python
import mlx.core as mx
from mlx_lm import load, generate

model, tokenizer = load("mourningdove/zk-auditor")

messages = [
    {"role": "system", "content": "You are a ZK proof security auditor specializing in Circom."},
    {"role": "user", "content": "Audit this circuit for vulnerabilities: template Test() { signal input a; signal output b; b <-- a * 3; }"}
]
prompt = tokenizer.apply_chat_template(messages, tokenize=False, add_generation_prompt=True)


response = generate(
    model, 
    tokenizer, 
    prompt=prompt, 
    max_tokens=300,
    sampler=lambda x: mx.argmax(x, axis=-1)
)
print(response)
```

## Usage (Transformers)

Coming soon. The plan is to provide a working Transformers example in the next version.

## A Note on `apply_chat_template`

This model was fine-tuned using Qwen's ChatML format, so you must apply the correct chat template when running inference in Python. Without it, the model receives a prompt structure that it was never trained on and will produce incorrect output.

`apply_chat_template` formats your messages into the ChatML structure the model expects. If your `tokenizer_config.json` is missing the `chat_template` key, inference can fail silently. Use the following to verify that your chat template matches the base model's template:

```python
from transformers import AutoTokenizer

ref = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-Coder-1.5B-Instruct")
tokenizer = AutoTokenizer.from_pretrained("mourningdove/zk-auditor")
print(ref.chat_template == tokenizer.chat_template)
```

## Training

The initial model we use is [Qwen2.5-Coder-1.5B-Instruct](https://huggingface.co/Qwen/Qwen2.5-Coder-1.5B-Instruct). A smaller model (1.5B parameters) is selected so the progression of our fine-tuned model will be more apparent when additional training data is added. This model can be downloaded from Hugging Face using their CLI tools.

```
hf download Qwen/Qwen2.5-Coder-1.5B-Instruct
```

We can train the model to obtain the adapter weights.

```
mlx_lm.lora --config config_v00.yaml --train --iters 10
```

Fuse the adapter weights to the base model:

```
mlx_lm.fuse \
  --model Qwen/Qwen2.5-Coder-1.5B-Instruct \
  --adapter-path adapters/v00/ \
  --save-path fused/
```

Verify the model works:

```
mlx_lm.generate \
  --model fused/ \
  --prompt "Audit this Circom circuit for vulnerabilities: template Test() { signal input a; signal output b; b <-- a * 2; }"
```

We can upload to our Hugging Face repository with the following:

```
hf upload mourningdove/zk-auditor fused --repo-type model
```

## Roadmap

- **v0.1** — 40–50 examples, real findings from public audits, broader pattern coverage
- **v0.2** — 100+ examples, held-out evaluation set, honest benchmark results


## License

Model weights: Apache 2.0 (inherited from base model)

