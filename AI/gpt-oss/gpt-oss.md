# What is GPT-OSS?
`GPT-OSS` is OpenAIs new "open-weight" model family -- that is OpenAI released the actual trained parameters so you can `download`, `run`, and `customize` it entirely on your on hardware without calling OpenAPI's API.

Here's what that offers you as a personal "AI PC" user:

1. Local, Private AI

* Runs fully offline -- no inernet needed once the model is download to your Mac/PC
* Your prompts and data `never leave your machine`
* Great for coding assistance, research, writing, or sensitive projects

2. Two Model Sizes

* GPT-OSS-20B (~21 B parameters, 3.6 B active per token)
  * Fits on 16-24 GB Macs in quantized form
  * Optimized for edge devices and local devices
* GPT-OSS-120B (~117 B total, ~5.1 B active per token)
  * Comparable reasoning to OpenAI's o4-mini, but needs an 80 GB GPU -- not laptop friendly

3. Good REasoning + Efficiency

* Mixture-of-Experts (MoE) architecture = only a fraction of expers activate per token -> `lower memory & compute demand` than dense model of similar size.
* Benchmarks show `20B holds` its on against mid-range open models for reasoning, code, and general chat.
