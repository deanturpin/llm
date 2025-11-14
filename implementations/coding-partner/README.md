# Coding Partner Implementation

This is a **starter template** for building a Claude Code-style agentic coding assistant.

## What You'll Build

An AI coding partner that can:
- Read and understand your codebase
- Write and edit code files
- Execute git commands
- Run tests and builds
- Explain code and suggest improvements

## Architecture Overview

See the [full use case guide](../../use-cases/coding-partner.md) for detailed architecture and feature descriptions.

## Prerequisites

- A VPS (recommended: Hetzner CCX33, 32GB RAM, £44/month)
  - Or home server with good CPU/RAM
- Docker and Docker Compose installed
- Good command line knowledge
- Understanding of tool-calling LLMs

## Quick Start

1. **Clone this repository**
   ```bash
   git clone https://github.com/yourusername/llm-guide
   cd llm-guide/implementations/coding-partner
   ```

2. **Create your private implementation repo**
   ```bash
   mkdir ~/my-coder-private
   cp -r . ~/my-coder-private/
   cd ~/my-coder-private
   git init
   ```

3. **Follow the implementation steps below**

## Implementation Steps

### Step 1: Core Infrastructure

Set up:
- Ollama with code-specific model (DeepSeek Coder, CodeLlama)
- Qdrant for codebase RAG
- FastAPI with tool-calling framework

### Step 2: Codebase Indexing

Implement:
- Recursive file scanning
- Language detection
- AST parsing (optional but powerful)
- Code chunking strategies
- Embedding and indexing in Qdrant

### Step 3: Tool Framework

Implement tools:
- File operations (read, write, edit)
- Git operations (status, diff, commit, push)
- Command execution (tests, builds)
- Search operations (grep, find)

### Step 4: Agent Loop

Implement:
- Multi-step reasoning
- Tool selection and execution
- Error handling and retry logic
- Context management

### Step 5: Safety and Sandboxing

Implement:
- Command whitelisting
- Dry-run mode
- Git integration (never force push, etc.)
- File system boundaries

## Technology Choices

### LLM Model
**Recommended**: DeepSeek Coder V2 16B or 33B
- Excellent code understanding
- Good tool use
- Strong debugging

**Alternatives**: CodeLlama 70B (quantised), Qwen2.5-Coder 32B

### For Tool Calling
Options:
1. Fine-tune on tool-use dataset
2. Use function-calling capable model
3. Prompt engineering with examples

### Codebase Embeddings
**Recommended**: CodeBERT or GraphCodeBERT
- Trained on code
- Understands syntax and semantics

## Resource Requirements

### Minimum
- 32GB RAM for 16B model (Q4)
- 4-8 vCPU
- £44/month (Hetzner CCX33)

### Recommended
- 64GB RAM for 33B model or multiple models
- 8+ vCPU
- GPU for faster inference (optional)

## Example Implementation

This is more complex than the PA. Key challenges:
- Tool-calling reliability
- Multi-step planning
- Error recovery
- Safety (don't delete important files!)

## Deployment Guide

Can run:
- **On VPS**: Access from anywhere
- **Locally**: Lower latency, no cloud costs
- **Hybrid**: Index on VPS, inference local

## Cost Estimate

- **VPS**: £44/month (CCX33)
- **Or local**: One-time hardware cost
- **Or hybrid**: £11/month (small VPS) + local hardware

## Next Steps

1. Read the [full use case guide](../../use-cases/coding-partner.md)
2. Consider: Do you really need this, or is Claude Code sufficient?
3. If building: Start with simple tool integration
4. Gradually add more sophisticated features

## Keep Your Data Private!

Your private implementation repo should contain:
- Your codebase (if indexing)
- Configuration and secrets
- Custom tools and workflows
