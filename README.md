# Lothar's Poor Man's AI Setup
- All local
- All free
- All Semi-autonomous

## Objective

A locally running semi-autonomous free and independent agent. Thus minimalized
and focussed just on coding tasks.

Note, the setup can be connected to API-LLMs.

My setup is a Debian Linux testing/sid (as base), having 128GB RAM and 20GB VRAM
GPU. The system will host a quantized 30B param LLM, dockerized. This will be
accessible by WebUI (browser), by neovim plugin, and/or in a semi-autonomous
agent using Aider.

# GPU: NVIDIA
Alternatively, running on pure CPU this setup will suffer badly. Alternatively,
connect it to AI APIs online and live with the consequences. In such case,
there's no need to set it up as "minimal" setup.

Make sure nvidia drivers are setup in the package manager, further install the `nvidia-container-toolkit`.
ref: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

### NVIDIA driver installation
Always when the kernel was updated, the nvidia modules need to be re-build!  
```
$ sudo apt update                                                               
```
                                                                             
Get sources and reinstall drivers  
```
$ sudo apt install -y linux-headers-$(uname -r)                                 
$ sudo apt install --reinstall nvidia-driver nvidia-kernel-dkms                 
                                                                                
(opt) probably not needed, but in case of trouble with docker - update docker toolkit packages
$ sudo apt install -y nvidia-container-toolkit                                  
$ sudo nvidia-ctk runtime configure --runtime=docker                            
$ sudo systemctl restart docker
```

### NVIDIA docker integration
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

# OLLAMA Container
The LLM environment, several LLMs can be installed. The LLM storage is persistent.
### Build OLLAMA container
First usage:
```
$ cd ./01_docker-ollama/
01$ docker build -t ollama-local .
01$ docker network create ollama-net
01$ docker run -d --name ollama --network ollama-net --gpus all -p 11434:11434 -v ollama-data:/root/.ollama ollama-local
```
(opt) Alternatively, do the CPU only version
```
01$ docker run -d --name ollama --network ollama-net -p 11434:11434 -v ollama-data:/root/.ollama ollama-local
```
In case remove a stale container (when re-started, but no network)
```
$ docker rm -f ollama
```
### Install LLMs for OLLAMA Container
Find available models here: https://www.ollama.com/  

[01/2026]: Given a focus on C/C++/Python/Rust for Zephyr RTOS, Linux kernel, U-boot with a limitation here of 20GB:
- Primary pick: Qwen3-Coder (quantized 30B) - best on benchmarked code tasks, huge context helps Linux/U-boot code.
- Alternative: DevStral-Small-2
- General pick: Mistral2
- Alternative: qwen3:14b

List installed LLMs
```
$ docker start ollama
$ docker exec ollama ollama list
NAME                       ID              SIZE      MODIFIED    
    mistral:7b                 6577803aa9a0    4.4 GB    2 weeks ago    
    DevStral-Small-2:latest    24277f07f62d    15 GB     5 weeks ago    
    qwen3-coder:30b            06c1097efce0    18 GB     5 weeks ago    
    qwen3:14b                  bdbd181c33f2    9.3 GB    5 weeks ago
```
Install LLM, note the in some cases billion params needs to be specified, e.g.
`qwen3:14b` or `qwen3-coder:30b`. Note, OLLAMA takes care of quantization, so in
case of this system 30 billion params are actually too much, anyway, OLLAMA
downloads the quantized version (if available), anyway it occupies 18GB disk
space.
```
$ docker exec -it ollama ollama pull qwen3-coder
```
The new model should appear in the above list. From webui, use the Model
Selector Dropdown to select a model. Or, re-run the docker command to build it
as default model.  

Check the LLM in local container and shell access
```
$ docker exec -it ollama bash
$ ollama run qwen3-coder:30b
```
or
```
$ docker exec -it ollama ollama run qwen3-coder:30b
```

### Start OLLAMA Container
```
$ docker start ollama
```
-> **OLLAMA** is reachable at: http://localhost:11434

# WebUI Container
The browser frontend to talk to the AI. Chats are not persistent over removeing container instance, if not explicitely saved!
### Build WebUI container
First usage:
```
$ cd ./02_docker-webui/
02$ $ docker build -t ollama-webui .
02$ docker run -d --name ollama-webui --network ollama-net -p 3000:8080 -e OLLAMA_BASE_URL=http://ollama:11434 ollama-webui
```
### Start WebUI Container
```
$ docker start ollama-webui
```
-> **WebUI** is reachable in browser: http://localhost:3000

# Application: Neovim
verify
```
curl http://localhost:11434/api/tags
```

### Neovim configuration
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
        model = "qwen3-coder:30b",
        host = "localhost",
        port = "11434",
      })
    end,
  },
})
```

### Neovim usage
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

# Application: Aider
Setup semi-autonomous AI-agent tool.

### Install Aider setup
```
$ sudo mkdir /opt/aider && sudo chown $(id -u):$(id -g) /opt/aider
$ cd /opt/aider
$ python3 -m venv aider-venv
$ . ./aider-venv/bin/activate
(aider-venv)$ pip install --upgrade pip setuptools wheel
(aider-venv)$ pip install aider-install
(aider-venv)$ aider-install
(aider-venv)$ $ aider --version
    aider 0.86.2
```

configure e.g. in .bashrc something like the following
```
...
export OPENAI_API_BASE=http://localhost:11434
```

### Aider Usage
due to the alias, do
```
$ aider-venv
(aider-venv)$ cd /data/z/github__docker__zephyr/docker/zephyrproject
(aider-venv)$ aider --model ollama/qwen3-coder:30b --no-show-model-warnings
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
Aider v0.86.2
Model: ollama/qwen3-coder:30b with whole edit format
Git repo: .git with 55,990 files
Warning: For large repos, consider using --subtree-only and .aiderignore
See: https://aider.chat/docs/faq.html#can-i-use-aider-in-a-large-mono-repo
Repo-map: using 4096 tokens, auto refresh
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
>  
```
Note, aider warns that `qwen3-coder:30b` is unknown to it, but fine to use anyway.

- Use '!' to do shell commands, e.g. `!ls -l` or `!pwd`
- Go through the code interactively, add files / folders (sub-section of codes) 
- `/add <file>` add files to the context
- `/drop` drop all added files

# Miscellaneous
## WebUI Configuration
What Ollama can do? Create derived models using Modelfiles:
- system prompts
- temperature
- context
- adapters (if supported)

Example Modelfile:
```
$ vi ./Modelfile
    FROM qwen3-coder:30b
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

## Commands and Usage 
cleanup
```
$ docker stop ollama-webui
$ docker stop ollama
```
clean persistence also
```
$ docker rm -f ollama
$ docker rm -f ollama-webui
```
clean, reestablish network
```
$ docker network create ollama-net
```
start and configure to use up to 19GB (of 20GB) more aggressive usage, using 8192 (if instable alternatively try 6144)
```
01$ docker run -d --name ollama --network ollama-net --gpus all --shm-size=32g -e OLLAMA_GPU_OVERHEAD=1024 -e OLLAMA_NUM_PARALLEL=1 -e OLLAMA_CONTEXT_LENGTH=8192 -p 11434:11434 -v ollama-data:/root/.ollama ollama-local
```
then in 02 start the webUI
```
02$ docker run -d --name ollama-webui --network ollama-net -p 3000:8080 -e OLLAMA_BASE_URL=http://ollama:11434 ollama-webui
```
configure:
- Context Length: 6144 (check this in the log)
- `Max Tokens`: 10000 
- `Temperature`: 0.2 – 0.3
- `Top_p`: 0.9
- `think (Ollama)`: Off (qwen3-code)
- `num_ctx (Ollama)`: 8192 [2048]

stability:
- Parallel Requests: 1 (check in log)
- Streaming: ON

## Sitting behind a firewall
The OLLAMA server can be accessed within the LAN, also VPN tunneled connects
work but in case need some forwarding setup.

In case tunnel the needed ports out, then on the client the following needs to
stay open:
```
$ ssh -N   -L 3000:<OLLAMA server IP>:3000   -L 11434:<OLLAMA server IP>:11434 user@<tunnel peer endpoint>
```
e.g the server is in a LAN having the IP 192.168.1.77/24, connecting to some
server (having forwarding, MASQUERADING and routing set), which is VPN endpoint
with IP, say, 192.168.7.3. Then the command on the local PC connecting to this
server remotely is
```
$ ssh -N   -L 3000:192.168.1.77:3000   -L 11434:192.168.1.77:11434   user@192.168.7.3
```

verify OLLAMA is accessible
```
curl http://localhost:11434/api/tags
```


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
