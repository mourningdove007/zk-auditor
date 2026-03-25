---
license: apache-2.0
base_model: Qwen/Qwen2.5-Coder-1.5B-Instruct
tags:
- zero-knowledge
- zk-proofs
- circom
- security
- circuit-constraints
- fine-tuned
- lora
language:
- en
pipeline_tag: text-generation
---

# ZK Constraint Auditor v0.0

A LoRA fine-tune of `Qwen2.5-Coder-1.5B-Instruct` specialized in identifying insufficient constraints in ZK proof circuits.

## Model description

This model is trained to answer one focused question: given a Circom circuit, is the constraint system sufficient to uniquely determine the values that matter? It identifies patterns where a malicious prover has degrees of freedom to substitute values that satisfy the circuit without representing a valid computation.

This is a v0.0 proof of concept. The model demonstrates the fine-tuning pipeline and constraint insufficiency reasoning on simple synthetic examples. It is not production-ready and will generalize poorly outside the patterns seen in training.

## Intended use

- First-pass triage of Circom circuits during security audits
- Educational tool for learning constraint insufficiency patterns
- Research baseline for ZK security tooling

## Vulnerability patterns

| Pattern | Description |
|---------|-------------|
Hint without recomposition | `<--` with no binding `===` |
Boolean check only | `x * (x-1) === 0` but `x` is never tied to the source |
Underdetermined system | 1 equation, 2 unknowns |
Weak final check | `a * b === 0` instead of `a === 0` |
Missing range constraint | bits decomposed but never bounded |
Unconstrained output | output assigned with `<--` not `<==` |
Partial constraint | only some iterations constrained in a loop |
Missing input validation | signal used before being constrained |
Reused signal across contexts | same signal constrained differently in two places |
Unconstrained intermediate | intermediate value computed but never tied to output |


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

## Training data

Fine-tuned on the [mourningdove/zk-constraint-data](https://huggingface.co/datasets/mourningdove/zk-constraint-data) dataset. These are synthetic, paired circuit examples covering 5 constraint insufficiency patterns. The full dataset is available in the [mourningdove007/zk-constraint-data repository](https://github.com/mourningdove007/zk-constraint-data) GitHub repository.

## Usage

```python
from transformers import AutoTokenizer, AutoModelForCausalLM

tokenizer = AutoTokenizer.from_pretrained("mourningdove/zk-auditor")
model = AutoModelForCausalLM.from_pretrained("mourningdove/zk-auditor")

prompt = """You are a ZK proof security auditor specializing in Circom.

Audit this circuit for vulnerabilities:

```circom
template Example() {
    signal input a;
    signal input b;
    signal output c;

    c <-- a * b;
}
```"""

inputs = tokenizer(prompt, return_tensors="pt")
outputs = model.generate(**inputs, max_new_tokens=300)
print(tokenizer.decode(outputs[0], skip_special_tokens=True))
```

## Limitations

- Trained on 15 simple synthetic examples — will not generalize to complex real-world circuits
- Circom only — does not cover Halo2, Arkworks, gnark, or Cairo
- May miss novel constraint patterns not seen in training
- Never use model output as the sole basis for a security finding — always verify manually

## Roadmap

- **v0.1** — 40–50 examples, real findings from public audits, broader pattern coverage
- **v0.2** — 100+ examples, held-out evaluation set, honest benchmark results

## Training

To test the training phase, we can run only ten iterations.
Once this is successful, we can remove the override so the config drives the iteration count.

```
mlx_lm.lora --config config_v00.yaml --train --iters 10
```

We can upload updates to our Hugging Face repository with the following:

`hf upload mourningdove/zk-auditor .`



## License

Model weights: Apache 2.0 (inherited from base model)

