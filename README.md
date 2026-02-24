# OpenGradient Plugin for Claude Code

A [Claude Code plugin](https://docs.anthropic.com/en/docs/claude-code/plugins) that provides expert guidance for the [OpenGradient Python SDK](https://docs.opengradient.ai) - a decentralized AI inference platform.

## What it does

When installed, this plugin adds the `opengradient` skill to Claude Code. Use it to get help writing correct, idiomatic code with the OpenGradient SDK, including:

- **Verified LLM inference** via TEE (Trusted Execution Environment)
- **x402 payment settlement** on Base Sepolia (on-chain receipts)
- **Multi-provider models** — OpenAI, Anthropic, Google, and xAI through a unified API
- **Streaming** and **tool calling** with full multi-turn agent loops
- **On-chain ONNX model inference** (alpha)
- **LangChain integration** for building ReAct agents
- **Digital twins** chat
- **Model Hub** — upload and manage models

## Installation

```bash
claude plugin add /path/to/claude-plugin
```

## Usage

Invoke the skill in Claude Code:

```
/opengradient-sdk write a streaming chat completion with tool calling
```

Or just describe what you want to build with OpenGradient — Claude will automatically use the skill when relevant.

## Project structure

```
.claude-plugin/
  plugin.json          # Plugin metadata
skills/
  opengradient/
    SKILL.md           # Skill definition and code patterns
    api-reference.md   # Detailed API reference
```

## Links

- [OpenGradient Documentation](https://docs.opengradient.ai)
- [OpenGradient Website](https://opengradient.ai)
