FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV OLLAMA_BASE_URL=http://ollama:11434

RUN pip install --no-cache-dir open-webui

EXPOSE 8080

## note: the following won't work, are old versions or anyway broken
#CMD ["open-webui", "--host", "0.0.0.0", "--port", "8080"]  ## no --host
#CMD ["open-webui", "--bind", "0.0.0.0:8080"]               ## no --bind
#CMD ["open-webui", "serve", "--bind", "0.0.0.0:8080"]      ## no --bind

## working:
CMD ["open-webui", "serve", "--port", "8080"]
