# Coding Partner Use Case

A Claude Code-style agentic coding assistant that can:
- Understand your codebase via RAG
- Write and edit code files
- Execute git commands
- Run tests and builds
- Provide architectural suggestions

## Why Build Your Own?

**Claude Code is excellent and you should use it!** But you might want your own if:
- Privacy concerns (don't want code sent to Anthropic)
- Cost optimisation (heavy usage gets expensive)
- Customisation (specific workflows, tools, integrations)
- Learning experience (understand how agentic systems work)
- Offline capability (work without internet)

## Honest Assessment

**Building this is hard.** Consider:
- Claude Code has years of R&D behind it
- Tool-calling needs extensive training/fine-tuning
- Multi-step reasoning is challenging
- Safety and reliability require careful design

**Recommendation**: Use Claude Code unless you have specific requirements that it doesn't meet.

## Architecture

```
┌─────────────────────────────────┐
│    IDE Extension or CLI         │
└────────────┬────────────────────┘
             │
    ┌────────▼─────────┐
    │  Agent Loop      │  ← Plans, executes tools, reflects
    │  (Orchestration) │
    └────────┬─────────┘
             │
    ┌────────┴─────────┐
    │                  │
┌───▼──────┐   ┌──────▼─────┐
│ Codebase │   │   Tools    │
│   RAG    │   │  Framework │
└───┬──────┘   └──────┬─────┘
    │                 │
    │        ┌────────┴────────┐
    │        │                 │
┌───▼────┐ ┌─▼──────┐  ┌─────▼────┐
│ Qdrant │ │  Bash  │  │   Git    │
│(vectors)│ │ (exec) │  │  (VCS)   │
└────────┘ └────────┘  └──────────┘
           │
      ┌────▼─────┐
      │  Ollama  │  ← DeepSeek Coder, CodeLlama
      │  (LLM)   │
      └──────────┘
```

## Key Components

### 1. Codebase RAG

Index your code for semantic search:
- Parse files, extract functions/classes
- Generate embeddings (CodeBERT)
- Store in vector database
- Retrieve relevant code for context

**Why?** LLMs have context limits. RAG lets the agent find relevant code across large codebases.

### 2. Tool Framework

Implement tools the agent can use:

**File Operations:**
- `read_file(path)` - Read file contents
- `write_file(path, content)` - Create/overwrite file
- `edit_file(path, old, new)` - Surgical edit

**Git Operations:**
- `git_status()` - Check repo state
- `git_diff()` - View changes
- `git_commit(message)` - Commit changes
- `git_push()` - Push to remote

**Command Execution:**
- `run_command(cmd)` - Execute shell command
- With safeguards (whitelist, sandboxing)

**Search Operations:**
- `grep_code(pattern)` - Search codebase
- `find_files(pattern)` - Find files by name

### 3. Agent Loop

Multi-step reasoning:
1. **Plan**: Break task into steps
2. **Act**: Execute tool calls
3. **Observe**: Examine results
4. **Reflect**: Adjust plan if needed
5. **Repeat**: Until task complete

### 4. Safety Mechanisms

Critical for production:
- **Command whitelisting**: Only allow safe commands
- **Dry-run mode**: Preview changes before applying
- **Git integration**: Never force push to main
- **Sandboxing**: Limit file system access
- **Confirmation prompts**: For destructive operations

## Implementation Challenges

### Challenge 1: Tool-Calling Reliability

Open-source models are **less reliable** at tool calling than Claude/GPT-4.

**Solutions:**
- Fine-tune on tool-use datasets
- Use structured output (JSON mode)
- Implement retry logic
- Provide clear tool schemas in prompts

### Challenge 2: Multi-Step Planning

Agents often:
- Get stuck in loops
- Forget original goal
- Make incorrect assumptions

**Solutions:**
- Implement step limit
- Track goal in every prompt
- Use explicit reflection steps
- Log and monitor behaviour

### Challenge 3: Context Management

Code context grows quickly:
- File contents
- Tool outputs
- Conversation history

**Solutions:**
- Aggressive context pruning
- Summarise old messages
- Use RAG to pull in only relevant code
- Implement memory systems

### Challenge 4: Safety

Easy to accidentally:
- Delete important files
- Commit broken code
- Expose secrets
- Corrupt git history

**Solutions:**
- Extensive testing in sandboxed environment
- Git-based safety (always commit before major changes)
- Secret detection (never commit .env files)
- Mandatory code review

## Model Selection

### Best for Coding: DeepSeek Coder V2

**DeepSeek Coder V2 16B** (recommended)
- Excellent code understanding
- Good instruction following
- Reasonable resource requirements (32GB RAM, Q4)

**DeepSeek Coder V2 33B** (if you have resources)
- Better reasoning
- Fewer errors
- Needs 64GB+ RAM or GPU

### Alternative: CodeLlama 70B

- Strong coding abilities
- Larger context window
- Needs quantisation (Q4) to fit

### Alternative: Qwen2.5-Coder 32B

- Very capable
- Good tool use
- Multilingual code support

## Resource Requirements

### Minimum
- **RAM**: 32GB (for 16B model, Q4)
- **CPU**: 8 cores
- **Disk**: 100GB+
- **VPS**: Hetzner CCX33 (£44/month)

### Recommended
- **RAM**: 64GB (for 33B model or multiple models)
- **CPU**: 16 cores
- **GPU**: RTX 3090 or better (much faster)
- **VPS**: Hetzner CCX63 or dedicated GPU instance

### Local Option
- Build a home server
- One-time cost vs monthly VPS
- Lower latency
- Full control

## Deployment Options

### Option A: Fully Remote
- Agent runs on VPS
- Access via web UI or CLI
- Always available
- Higher latency

### Option B: Fully Local
- Agent runs on your machine
- Lowest latency
- Works offline
- No cloud costs

### Option C: Hybrid
- Codebase RAG on VPS (always indexed)
- Agent inference locally (faster, private)
- Best of both worlds
- More complex setup

## Feature Comparison

| Feature | Claude Code | Self-Hosted |
|---------|-------------|-------------|
| Code understanding | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Tool use reliability | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| Multi-step reasoning | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| Setup time | 5 min | Days-weeks |
| Privacy | ⚠️ Sent to Anthropic | ✅ Fully local |
| Cost (heavy use) | £16+/month | £44-80/month |
| Customisation | Limited | ✅ Full control |
| Offline capable | ❌ No | ✅ Yes (local) |

## Implementation Roadmap

### Phase 1: Basic Tools (1-2 weeks)
- Implement file read/write
- Basic git operations
- Command execution with whitelist
- Simple CLI interface

### Phase 2: RAG Integration (1-2 weeks)
- Code indexing pipeline
- Semantic search
- Context retrieval
- Integration with tools

### Phase 3: Agent Loop (2-3 weeks)
- Plan-Act-Observe-Reflect cycle
- Multi-step execution
- Error handling and retry
- Conversation history

### Phase 4: Safety and Polish (1-2 weeks)
- Comprehensive testing
- Safety mechanisms
- Dry-run mode
- User experience improvements

**Total time**: 6-9 weeks for basic working version

## Cost-Benefit Analysis

### Costs
- **Development time**: 6-9 weeks (your time)
- **Infrastructure**: £44-80/month
- **Maintenance**: Ongoing

### Benefits
- **Privacy**: Code never leaves your infra
- **Customisation**: Build exactly what you need
- **Learning**: Deep understanding of agentic systems
- **Control**: No rate limits, no API changes

**Worth it if**:
- You have privacy requirements
- You want deep customisation
- You enjoy building systems
- You have the time

**Not worth it if**:
- Claude Code meets your needs
- Time is more valuable
- You want something that "just works"

## Recommended Approach

### If You're Serious About Building This:

1. **Start small**: Build a simple tool-calling system first
2. **Use Claude Code**: Learn how it works by using it extensively
3. **Study open source**: Look at AutoGPT, MemGPT, LangChain agents
4. **Prototype rapidly**: Don't aim for perfection initially
5. **Test extensively**: In sandboxed environments
6. **Iterate**: Gradually improve reliability

### If You Just Want a Coding Assistant:

**Use Claude Code.** Seriously, it's excellent.

## Implementation Guide

See [../implementations/coding-partner/](../implementations/coding-partner/) for:
- Starter template
- Tool framework example
- Agent loop skeleton
- Safety guidelines

## Learning Resources

- **Tool Calling**: OpenAI function calling docs, Anthropic tool use guide
- **Agent Systems**: LangGraph, AutoGPT architecture
- **Code Understanding**: Tree-sitter for parsing, CodeBERT for embeddings
- **Safety**: OWASP top 10, secure coding practices

## FAQs

**Q: Can I just use LangChain?**
A: Yes! LangChain provides agent primitives. But you'll still need to handle code-specific challenges.

**Q: What about open-source alternatives like Continue or Aider?**
A: These are great starting points! Consider contributing or forking instead of building from scratch.

**Q: Can I fine-tune a model on my codebase?**
A: Yes, but RAG is usually better. Fine-tuning helps with coding style, RAG with codebase knowledge.

**Q: How do I handle secrets (API keys, passwords)?**
A: Use environment variables, never commit `.env` files, implement secret detection in pre-commit hooks.

## Conclusion

Building a coding partner is **challenging but rewarding**. Consider your goals:

- **Learning**: Great project
- **Privacy**: Valid requirement
- **Productivity**: Claude Code is probably better (for now)

If you proceed, start small and iterate!
