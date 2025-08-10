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

## Make olloama accessible from within home network
we have to tell ollama to listen to 0.0.0.0, which mean get all traffic from anywhere

``` bash
#!/usr/bin/env bash

# check if ollama is listening
lsof -iTCP:11434 -sTCP:LISTEN 2>&1 >/dev/null
if [[ $? == 0 ]]; then
   echo ollama is already running!!
   exit 0
fi

# allow access ollama from anywhere
export OLLAMA_HOST=0.0.0.0:11434
OLLAMA_HOME=$(brew --prefix ollama)
OLLAMA_BIN="$OLLAMA_HOME/bin"

nohup $OLLAMA_BIN/ollama serve 2>&1 > /dev/null &
```
The above script will start ollama and make it accessible from anywhere in your home network. You can save this script as `start-ollama.sh` and run it whenever you want to start ollama.

To check if ollama is running, run this command:
``` bash
curl -s http://localhost:11434/v1/models | jq

```
## Make gpt-oss model running
``` bash
curl http://xyz.local:11434/api/generate -d '{"model":"gpt-oss:20b","prompt":"ready","options":{"num_ctx":4096},"keep_alive":"2h"}'

```
The above curl command will send a json to a remote host that start `gpt-oss:20b` with context that take 4096 tokens, and stay alive for 2 hours

## Secure ollama

### Turn off accept remote connections from anywhere
previously we use localhost:0.0.0.0:11434 in order to allow traffic from any labtop in our home network, but this is not secure. We want to accept traffic from known host only. Here's how to do it:

* First, remove OLLAMA_HOST=0.0.0.0:11434
* Second, we will setup SSH reverse tunnel from ollama server to the client that we want to access ollama from. This way, we can access ollama from the client without exposing it to the internet. We use autossh to make sure the tunnel is always up.

      ``` bash
      autossh -f -M 20000 -N \
        -R 11434:localhost:11434 \
        -o ExitOnForwardFailure=yes \
        -o ServerAliveInterval=10 -o ServerAliveCountMax=1 -o TCPKeepAlive=yes \
        -o ConnectTimeout=5 -o ConnectionAttempts=9999 -o BatchMode=yes \
        hostname.local
      ```
  We use Mac mDns hostname to connect to the client for building up the tunnel. This way, we don't need to worry about IP address changes in our home network.

* from the client, we can simply run
  ``` bash
  curl -s http://localhost:11434/v1/models | jq
  ```
  to verify that we can access ollama from the client.

* To run gpt-oss model, we can simply run the same curl command as before, but this time we don't need to specify the host, because we are already connected to the ollama server through the SSH tunnel.

  ``` bash
  curl http://localhost:11434/api/generate -d '{"model":"gpt-oss:20b","prompt":"ready","options":{"num_ctx":4096},"keep_alive":"2h"}'
  ```

