# GPT-OSS internet access

Simple answer -- `GPT-OSS` cannot access internet as it is just the base language model.
It has no `built-in "agent" layer` and no native internet access

If you want it to query internet, you have to wrap it in your own agent framework

1. `Intercepts its output` for special commands
2. `Runs external tools/APIs` outside the model (e.g. "search for X")
3. `Feeds the retrieved text back` into this model as context

## How to access internet

* `LangChain / LlamaIndex` -> define a tool called `web_search` or `browse` and hook it to Bing, Brave, Google API, etc
* `Custom Ollama script` -> run a Python/Node shell scripts that monitors the LLM output for a trigger phrase and calls a search APIs
* `Local Agent frameworks` like `Open WebUI`, `Tabby Agent`, or `AnythingLLM` -< can integrate with GPT-OSS retrieval and search
