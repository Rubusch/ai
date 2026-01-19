# OLLAMA Environment
This will setup two docker containers communicating over a dockerized network.
One OLLAMA container and on top a WebUi container for a web front-end.
In this way, OLLAMA can be accessed directly in the shell or via API (for IDEs).
The optional WebUI provides a web front-end for browser access.

## Preparation
Make sure nvidia drivers are setup in the package manager, further install the `nvidia-container-toolkit`.
ref: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
```
$ sudo apt update
$ sudo apt install -y zstd
$ sudo apt install -y nvidia-container-toolkit
$ sudo nvidia-ctk runtime configure --runtime=docker
$ sudo systemctl restart docker
```
NVIDIA sanity checks for docker container
```
$ docker info | grep -i nvidia
$ docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

## Build OLLAMA Container
```
$ cd ./01_docker-ollama/
01$ docker build -t ollama-local .
```
create the network
```
01$ docker network create ollama-net
```

## Start OLLAMA Container
```
01$ docker run -d --name ollama --network ollama-net --gpus all -p 11434:11434 -v ollama-data:/root/.ollama ollama-local
```
(opt) alternatively, do the CPU only version
```
01$ docker run -d --name ollama --network ollama-net -p 11434:11434 -v ollama-data:/root/.ollama ollama-local
```

## Build WebUI Container
```
$ cd ./02_docker-webui/
02$ $ docker build -t ollama-webui .
```

## Start WebUI Container
```
02$ docker run -d --name ollama-webui --network ollama-net -p 3000:8080 -e OLLAMA_BASE_URL=http://ollama:11434 ollama-webui
```
then in browser
```
http://localhost:3000
```

# LLM
## LLM Download
find available models here:
https://www.ollama.com/

[01/2026]: Given a focus on C/C++/Python/Rust for Zephyr RTOS, Linux kernel, U-boot with a limitation here of 20GB:
- Primary pick: DevStral-Small-2 - strong code base reasoning + agentic tools + fits your GPU.
- Alternative powerful option: Qwen3-Coder (quantized 30B/14B) - best on benchmarked code tasks, huge context helps Linux/U-boot code.
- Good fallback: Qwen2.5-Coder for lighter local tasks or quick iterations.

list installed LLMs
```
$ docker exec ollama ollama list
    NAME                       ID              SIZE      MODIFIED     
    DevStral-Small-2:latest    24277f07f62d    15 GB     6 hours ago     
    qwen3-coder:30b            06c1097efce0    18 GB     6 hours ago     
    qwen2.5-coder:14b          9ec8897f747e    9.0 GB    17 hours ago    
    qwen3:14b                  bdbd181c33f2    9.3 GB    18 hours ago
```

```
$ docker start ollama
$ docker exec -it ollama ollama pull qwen3:14b
```
or, for particularly owned LLMs
```
$ docker exec -it ollama ollama pull freehuntx/qwen3-coder:14b
```
The new model should appear in the above list. From webui, use the Model Selector Dropdown to select a model.

## LLM Usage
Start local container and shell access
```
$ docker exec -it ollama bash
$ ollama run freehuntx/qwen3-coder:14b
```
or (TODO verify)
```
$ docker exec -it ollama ollama run freehuntx/qwen3-coder:14b
```
then in browser
```
http://localhost:3000
```
API access
```
http://localhost:11434
```

# OLLAMA
## OLLAMA Configuration
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

# Neovim
verify
```
curl http://localhost:11434/api/tags
```

installation
```
$ sudo apt install -y neovim
$ mkdir -p ~/.config/nvim
```
setup config (in case paste with SHIFT+CTRL+v)
- install lazy.nvim plugin manager
- install gen.nvim and configure it to talk to ollama
```
$ nvim ~/.config/nvim/init.lua
-- bootstrap lazy.nvim
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not vim.loop.fs_stat(lazypath) then
  vim.fn.system({
    "git",
    "clone",
    "--filter=blob:none",
    "https://github.com/folke/lazy.nvim.git",
    "--branch=stable",
    lazypath,
  })
end
vim.opt.rtp:prepend(lazypath)

-- plugins
require("lazy").setup({
  {
    "David-Kunz/gen.nvim",
    config = function()
      require("gen").setup({
        model = "freehuntx/qwen3-coder:14b",
        host = "localhost",
        port = "11434",
      })
    end,
  },
})
```
start neovim, type `:Gen` to access the menu
```
$ nvim
:Gen
```
test neovim on code, open a code file
```
$ nvim main.c
```
mark some lines
```
v
```
type `:` opens `:'<,'>` then type `Gen`, ENTER
```
:'<,'>Gen
```
- select `1` for `Ask`
- type: `explain this code`

# Troubleshooting
```
$ docker logs ollama-webui
```
in case
```
$ docker ps -a
...
```
see if the container is around, in case remove it before rebuilding
```
$ docker rm ollama-webui
$ docker build -t ollama-webui .
```
