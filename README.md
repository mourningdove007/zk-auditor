# ZK Constraint Auditor v0.1

A LoRA fine-tune of `Qwen2.5-Coder-1.5B-Instruct` specialized in identifying insufficient constraints in ZK proof circuits. Fine-tuned on data from the [mourningdove007/zk-constraint-data](https://github.com/mourningdove007/zk-constraint-data) GitHub repository. The model is available on [HuggingFace](https://huggingface.co/mourningdove/zk-auditor). Versions prior to `v1.0` will likely miss simple issues as the training data is still small. 


## Intended use

- First-pass triage of Circom circuits during security audits
- Educational tool for learning constraint insufficiency patterns
- Research baseline for ZK security tooling

## Configuration details Version 0.1

| Parameter | Value |
|-----------|-------|
| Base model | Qwen2.5-Coder-1.5B-Instruct |
| Method | LoRA (MLX) |
| LoRA layers | 8 |
| Training examples | 124 |
| Validation examples | 25 |
| Iterations | 175 |
| Learning rate | 2e-5 |


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
    sampler=lambda x: mx.random.categorical(x * (1/0.7)),
)
print(response)
```


## A Note on `apply_chat_template`

This model was fine-tuned using Qwen's ChatML format. 
`apply_chat_template` formats your messages into the ChatML structure the model expects:

```python
messages = [
    {"role": "system", "content": "You are a ZK proof security auditor specializing in Circom."},
    {"role": "user", "content": "Audit this circuit for vulnerabilities:\n\n```circom\n...\n```"}
]
prompt = tokenizer.apply_chat_template(messages, tokenize=False, add_generation_prompt=True)
```
The chat template is inherited from the base model (`Qwen2.5-Coder-1.5B-Instruct`).
Templates can be viewed using the transformers library.

```
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-Coder-1.5B-Instruct")
print(tokenizer.chat_template)
```


## Training

The initial model we use is [Qwen2.5-Coder-1.5B-Instruct](https://huggingface.co/Qwen/Qwen2.5-Coder-1.5B-Instruct). A smaller model (1.5B parameters) is selected so the progression of our fine-tuned model will be more apparent when additional training data is added. This model can be downloaded from Hugging Face using their CLI tools.

```
hf download Qwen/Qwen2.5-Coder-1.5B-Instruct
```

We can train the model to obtain the adapter weights. 
The parameters are configured in `config_v01.yaml`.

```
mlx_lm.lora --config config_v01.yaml --train \
2>&1 | tee training_log.txt
```

The best recorded loss against the validation data occurred at iteration 175. 
Afterwards, overtraining seems to occur as the validation loss becomes inconsistent and does not decrease beyond the recorded loss at iteration 175.
Setting `save_every: 25` in `config_v01.yaml` saves a checkpoint adapter every 25 iterations.
The adapter weights fused to the base model will be weights recorded at the iteration witnessing the smallest recorded validation loss.

```
cat adapters/v01/0000175_adapters.safetensors > adapters/v01/adapters.safetensors
```

Fuse the adapter weights to the base model:

```
mlx_lm.fuse \
  --model Qwen/Qwen2.5-Coder-1.5B-Instruct \
  --adapter-path adapters/v01/ \
  --save-path fused/
```

Verify the model works:

```
mlx_lm.generate \
  --model fused/ \
  --max-tokens 300 \
  --temp 0.7 \
  --prompt "Audit this Circom circuit for vulnerabilities: template Test() { signal input a; signal output c; c <-- a * 2; }"
```

Since this model is only trained on 124 datapoints, the model still cannot identify the issue with the circom circuit. 
Output can be expected to look like the following:

```
The Circom circuit is correctly implemented: it takes an input `a`, squares it (`a * 2`), and outputs the result `c`. The witness `c <-- a * 2` is a primitive witness built-in Circom, which computes the witness `c` from the input `a` on-demand.

The circuit is correctly constrained and does not have any implicit witness creation, so it is free of any logic bugs.
```

We can upload to our Hugging Face repository with the following:

```
hf upload mourningdove/zk-auditor fused --repo-type model
```

## License

Model weights: Apache 2.0 (inherited from base model)

