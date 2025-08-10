# How to configure Mac M4 airbook as minimal AI PC

This is a minimal setup that will make Mac Air book as AI PC. Here's the
requirements:

* Apps: iTerm2, Nevoim, Safari, and Ollama
* Power & sleep: System Settings -> Battery -> `Prevent sleeping on power`; set display sleep short, `computer sleep never` when plugined in
* Spotlight/Time Machine: exclude your models dir so macOS doesn't trash.
  * Splotlight: System Settings -> Siri & Spotlight -> `Privacy` -> add ~/Models
  * Time Machine: Settings -> `Options` -> exclude ~/Models
* Thermals: run on power; a stand + airflo helps the Air avoid throttling

## Install bits
``` bash
# Homebrew + basics
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install neovim git ripgrep fd
brew install --cask iterm2

# Ollama
brew install ollama
brew services start ollama  # auto-start in background
```

## Folder layout

``` bash
mkdir -p ~/Models ~/AI/{logs,work}
```
Keep one big model in ~/Models

## Ollama modelfile (good for defaults 24GB)

``` dockerfile
FROM /Users/$USER/Models/gpt-oss-20b.Q5_K_M.gguf

PARAMETER num_ctx 8192
PARAMETER num_batch 640
PARAMETER temperature 0.7
PARAMETER top_p 0.9
PARAMETER repeat_penalty 1.1

```

Build & run

``` bash
ollama create gptoss20b -f ~/Models/Modelfile
ollama run gptoss20b
```
