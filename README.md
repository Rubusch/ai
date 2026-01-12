# OLLAMA Environment
- Running on Debian w/ nvidia-driver (compiled LKM) installed
- Qwen3-coder:14B for 16GB VRAM
[- Qwen3-coder:32B for 24GB VRAM]

## Build
```
$ docker build -t ollama-qwen .
```

## Usage
NVIDIA, with auto-detect CUDA
```
$ docker run -d \
  --name ollama \
  --gpus all \
  -p 11434:11434 \
  -v ollama-data:/root/.ollama \
  ollama-qwen
```

CPU only
```
$ docker run -d \
  --name ollama \
  -p 11434:11434 \
  -v ollama-data:/root/.ollama \
  ollama-qwen
```

## LLM Download
```
$ docker exec -it ollama ollama pull qwen3-coder:14b
```

## LLM Usage
```
$ docker exec -it ollama ollama run qwen3-coder:14b
```

## Ollama Finetune
What Ollama can do? Create derived models using Modelfiles:
- system prompts
- temperature
- context
- adapters (if supported)

Example Modelfile:
```
$ vi ./Modelfile
    FROM qwen3-coder:14b
    PARAMETER temperature 0.2
    SYSTEM "You are a strict C++ code reviewer."
```
Then:
```
$ ollama create qwen3-coder-cpp -f Modelfile
```
This is not training. It’s configuration.

## Ollama Fine-Training
Real fine-tuning workflow (correct one)

1. Fine-tune externally using:
 - HuggingFace + PEFT
 - QLoRA / LoRA
2. Export: GGUF or compatible format
3. Import into Ollama:
```
$ ollama create my-qwen -f Modelfile
```
If someone tells you Ollama does end-to-end fine-tuning, they are confused.
