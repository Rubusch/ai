FROM debian:12-slim

ENV DEBIAN_FRONTEND=noninteractive
ENV OLLAMA_HOST=0.0.0.0
ENV OLLAMA_MODELS=/root/.ollama

# Install dependencies
RUN apt-get update && apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    zstd \
    libstdc++6 \
    && rm -rf /var/lib/apt/lists/*

# Install Ollama
RUN curl -fsSL https://ollama.com/install.sh | sh

# Expose Ollama API
EXPOSE 11434

# Persist models
VOLUME ["/root/.ollama"]

ENTRYPOINT ["ollama"]
CMD ["serve"]
