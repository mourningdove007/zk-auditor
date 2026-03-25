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

## Prerequisites

The initial model we use is [Qwen2.5-Coder-1.5B-Instruct](https://huggingface.co/Qwen/Qwen2.5-Coder-1.5B-Instruct). A smaller (1.5B parameters) model is selected so the progression of our fine-tuned model will be more apparent when additional training data is added. This model can be downloaded from HuggingFace using their CLI tools.

```
hf download Qwen/Qwen2.5-Coder-1.5B-Instruct
```

We use [MLX](https://github.com/ml-explore/mlx-lm) for fine-tuning an existing model. This can be installed with

```
pip install -U mlx-lm
```

## Training

To test the training phase, we can run only ten iterations.
Once this is successful, we can remove the override so the config drives the iteration count.

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

Verify the model works (for lower versions, the answer will likely be erroneous due to the small amount of training data used):

```
mlx_lm.generate \
  --model fused/ \
  --prompt "Audit this Circom circuit for vulnerabilities: template Test() { signal input a; signal output b; b <-- a * 2; }"
```


We can upload to our Hugging Face repository with the following:

```
hf upload mourningdove/zk-auditor fused/ --repo-type model
```


## License

Model weights: Apache 2.0 (inherited from base model)

