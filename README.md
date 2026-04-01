# Euclid — open-source replica of Claude Code.
> **Euclid** is a self-hosted, open-source replica of Claude Code. It runs on any Ollama model — locally or on a cloud GPU. It is a full agentic coding runtime: it plans, executes shell commands, reads and edits files, manages memory, orchestrates sub-agents, streams responses, and persists everything to SQLite. This document is a complete build manual. Every section is a direct instruction to an AI agent. Read every word. Execute every command. Build everything.

---

## Table of Contents

- [EUCLID — AI Agent Operating Manual](#euclid--ai-agent-operating-manual)
  - [Table of Contents](#table-of-contents)
  - [1. What Euclid Is](#1-what-euclid-is)
  - [2. Source Material Decoded](#2-source-material-decoded)
    - [2.1 src.zip — The Claude Code Agentic Backend](#21-srczip--the-claude-code-agentic-backend)
    - [2.2 everything-claude-code-main.zip — The Skills Ecosystem](#22-everything-claude-code-mainzip--the-skills-ecosystem)
    - [2.3 Inference\_Engineering.pdf — The GPU and Inference Science](#23-inference_engineeringpdf--the-gpu-and-inference-science)
    - [2.4 SenecaChat-master.zip — The Frontend and Server Patterns](#24-senecachat-masterzip--the-frontend-and-server-patterns)
  - [3. Tech Stack Decision](#3-tech-stack-decision)
  - [4. Project Scaffold](#4-project-scaffold)
  - [5. Inference Layer — Ollama on Cloud GPU](#5-inference-layer--ollama-on-cloud-gpu)
    - [5.1 Choosing Your Deployment Mode](#51-choosing-your-deployment-mode)
    - [5.2 Cloud GPU Setup — RunPod](#52-cloud-gpu-setup--runpod)
    - [5.3 Model Selection for Euclid](#53-model-selection-for-euclid)
    - [5.4 The Ollama Client](#54-the-ollama-client)
  - [6. The Core Agentic Loop](#6-the-core-agentic-loop)
  - [7. The Tool System](#7-the-tool-system)
    - [7.1 The Tool Interface](#71-the-tool-interface)
    - [7.2 The Tool Runner](#72-the-tool-runner)
    - [7.3 Core Tool Implementations](#73-core-tool-implementations)
  - [8. Session State and QueryEngine](#8-session-state-and-queryengine)
  - [9. Database Schema](#9-database-schema)
  - [10. The API Server](#10-the-api-server)
  - [11. Context Management and Compaction](#11-context-management-and-compaction)
    - [11.1 Token Counting](#111-token-counting)
    - [11.2 Tool Result Budget](#112-tool-result-budget)
    - [11.3 Auto-Compact](#113-auto-compact)
  - [12. The Skills System](#12-the-skills-system)
  - [13. Multi-Agent and Task System](#13-multi-agent-and-task-system)
    - [13.1 Built-in Agent Roster](#131-built-in-agent-roster)
    - [13.2 Multi-Agent Orchestration](#132-multi-agent-orchestration)
  - [14. BM25 RAG and Memory](#14-bm25-rag-and-memory)
  - [15. Permission and Safety Layer](#15-permission-and-safety-layer)
  - [16. The Frontend](#16-the-frontend)
  - [17. Auth, Rate Limiting, Circuit Breaker](#17-auth-rate-limiting-circuit-breaker)
    - [17.1 Auth](#171-auth)
    - [17.2 Rate Limiting](#172-rate-limiting)
    - [17.3 Circuit Breaker](#173-circuit-breaker)
  - [18. Observability and Evals](#18-observability-and-evals)
  - [19. Production Deployment on Cloud](#19-production-deployment-on-cloud)
    - [19.1 Cloud Architecture](#191-cloud-architecture)
    - [19.2 Nginx Configuration](#192-nginx-configuration)
    - [19.3 PM2 Process Management](#193-pm2-process-management)
    - [19.4 GPU Pod Setup (vLLM for Multi-User)](#194-gpu-pod-setup-vllm-for-multi-user)
    - [19.5 Environment Variables](#195-environment-variables)
  - [20. Full File Tree](#20-full-file-tree)
  - [21. Step-by-Step Build Commands](#21-step-by-step-build-commands)
  - [Appendix A — Reflexion Implementation](#appendix-a--reflexion-implementation)
  - [Appendix B — System Prompt Architecture](#appendix-b--system-prompt-architecture)

---

## 1. What Euclid Is

Euclid is a backend agentic runtime that wraps any Ollama model and gives it the same capability surface as Claude Code. The model is treated as a planner. Euclid is the policy-enforcing executor. Every tool the model can call goes through a schema-validated, permission-checked, result-capped, streaming-capable execution layer. The model never touches the filesystem, network, or shell directly. It requests actions through a typed tool surface and receives structured results.

The name Euclid reflects the project's purpose: rigorous, axiomatic construction from first principles. Just as Euclid built all of geometry from five postulates, this system builds a full coding agent from five components: the agentic loop, the tool registry, the state store, the inference engine, and the permission model.

Euclid is better than Claude Code in these specific ways: it runs locally or on your own cloud GPU at near-zero marginal cost, it gives you full transparency into every prompt, tool call, and memory operation, it uses the SenecaChat frontend which is battle-tested for long agentic sessions, and it adds Reflexion (structured self-critique on failure), BM25 RAG, multi-namespace memory, multi-agent orchestration, and heartbeat probes — none of which exist in the stock Claude Code UI.

---

## 2. Source Material Decoded

Before building a single line, you must understand what the three source archives contain. This section decodes each one completely.

### 2.1 src.zip — The Claude Code Agentic Backend

This archive is the actual source of the Claude Code CLI's agentic core. It is approximately 1,902 TypeScript files organized under `src/`. The files you must understand deeply are the following.

**`src/query.ts`** is the most important file in the entire codebase. It implements the main agentic while-loop. The function `queryLoop` runs in an `AsyncGenerator` that `yield`s `StreamEvent`, `Message`, and `ToolUseSummaryMessage` values. Each iteration of the loop does this in order: it prepares the message history (applying compaction, snipping, tool result budgets), calls the model with streaming, collects all `ToolUseBlock` items from the assistant's response, runs those tools concurrently via `runTools` or `StreamingToolExecutor`, collects `toolResults`, appends everything to the message history, and then `continue`s the loop for another iteration. The loop exits cleanly only when `needsFollowUp` is false — meaning the model produced a response with no tool calls. Every edge case is handled explicitly: abort signals, max-turn limits, prompt-too-long recovery (reactive compact), max-output-token recovery with injected continuation messages, stop hooks that can block continuation, token budget enforcement, and model fallback on rate limit.

The most critical insight from `query.ts` is the recovery message injected when `max_output_tokens` is hit. The exact wording is: "Output token limit hit. Resume directly — no apology, no recap of what you were doing. Pick up mid-thought if that is where the cut happened. Break remaining work into smaller pieces." You will replicate this exact recovery pattern in Euclid, adapted for Ollama's response format.

**`src/Tool.ts`** defines the complete `Tool` type interface. Every tool in the system implements this interface. The key methods are: `call()` which is the actual execution logic, `checkPermissions()` which returns `allow`, `ask`, or `deny`, `validateInput()` which runs before permissions, `prompt()` which returns the tool's description for the system prompt, `isConcurrencySafe()` which controls whether the tool can run in parallel with others, `isReadOnly()`, `isDestructive()`, and `mapToolResultToToolResultBlockParam()` which formats the result for the API. The `buildTool()` factory function fills in safe defaults — `isConcurrencySafe` defaults to `false` (assume not parallel-safe), `isReadOnly` defaults to `false` (assume writes), `checkPermissions` defaults to `allow`. You will implement a simplified but faithful version of this interface in JavaScript.

**`src/QueryEngine.ts`** is the session manager. It owns one conversation's entire lifecycle. It holds `mutableMessages` (the growing message array), `totalUsage` (cumulative token counts across all turns), `readFileState` (an LRU cache of file read hashes used for deduplication), and `abortController` (the signal that cancels in-flight requests). The `submitMessage()` method is what the API calls when the user sends a message. It assembles the full `ToolUseContext`, calls `query()`, drains the async generator, and emits SDK-formatted output. You will replicate this as a `QueryEngine` class in your backend.

**`src/Task.ts`** defines background tasks. Task types are: `local_bash` (shell commands), `local_agent` (in-process sub-agents), `remote_agent`, `in_process_teammate`, `local_workflow`, `monitor_mcp`, and `dream`. Each task has a unique ID with a type prefix (`b` for bash, `a` for agent, etc.), a status (`pending`, `running`, `completed`, `failed`, `killed`), and an output file path where streaming output is written to disk. The `generateTaskId()` function uses 8 random bytes from a 36-character alphanumeric alphabet (case-insensitive safe) producing 2.8 trillion combinations to resist brute-force symlink attacks.

**`src/tools.ts`** is the tool registry. It imports all tool implementations and exports them as a flat array. The registry is the single source of truth for which tools the model can see. Tools are filtered by `isEnabled()` before being passed to the model. Some tools are marked `shouldDefer: true` meaning they are not shown in the initial prompt — the model must call `ToolSearch` first to discover them.

**`src/commands.ts`** is the slash-command registry. Slash commands (like `/clear`, `/compact`, `/resume`) are human-invoked and run outside the model loop. They are different from tools, which are model-invoked.

### 2.2 everything-claude-code-main.zip — The Skills Ecosystem

This archive contains a collection of SKILL.md files organized as `.agents/skills/<skill-name>/SKILL.md`. Each skill is a structured markdown document that an AI reads before performing work in a specific domain. The skill system is what makes the agent domain-aware without fine-tuning. When the model needs to design an API, it reads the `api-design` skill. When it needs to write a backend service, it reads `backend-patterns`. When it needs to set up end-to-end tests, it reads `e2e-testing`.

The key insight from the skills architecture is the `agents/openai.yaml` file inside each skill directory. This YAML file configures a lightweight OpenAI-compatible sub-agent that is specialized for that skill. Each agent has its own model, system prompt, and temperature setting. This is how multi-agent specialization works without training separate models.

The skill system is composed of these components: a skill loader that scans the `.agents/skills/` directory, a skill selector that picks the most relevant skill(s) for a given task (using keyword matching on the SKILL.md descriptions), and a skill injector that prepends the skill content to the system prompt when that skill is active. You will build this entire pipeline in Euclid.

### 2.3 Inference_Engineering.pdf — The GPU and Inference Science

This 313-page book by Philip Kiely (Baseten Books, 2026) is the authoritative reference for running open models in production. Every architectural decision about Euclid's inference layer comes from this book. The following concepts are the ones you must implement or account for.

**KV Cache** (Chapter 5.3): The key-value cache stores the attention computations from previous tokens so they don't need to be recomputed. Ollama manages this automatically, but you must structure your system prompts to maximize cache re-use — identical system prompt prefixes across turns hit the cache; any change busts it. This is why Euclid's system prompt has a static "base" section and a dynamic "context" section that is appended separately.

**Quantization** (Chapter 5.1): Running a 70B model at FP16 requires approximately 140 GB of VRAM. Quantizing to Q4_K_M reduces this to about 40 GB, making it runnable on a single A100-80GB or two A6000-48GB GPUs. For Euclid, you will use Q4_K_M quantization for the default model. The quality tradeoff is minimal for coding tasks because the weights and activations are the least sensitive components to quantization. Only attention layers are high-risk; Ollama handles the precision decisions automatically for each format.

**Speculative Decoding** (Chapter 5.2): A small draft model proposes 4-8 tokens which the larger target model then validates in one forward pass. When all draft tokens are accepted, this is a free 4-8x speedup. Ollama 0.5+ supports speculative decoding via the `num_predict` parameter paired with a draft model. For Euclid, you will use `qwen2.5-coder:1.5b` as the draft model when serving `qwen2.5-coder:32b` as the main model.

**Prefill vs Decode** (Chapter 5.5): Prefill is the compute-bound phase where the system prompt and user message are processed in one forward pass. Decode is the memory-bandwidth-bound phase where one token is generated per forward pass. For agentic workloads, prefill can be 50-80% of total request time on long conversations. This is why context compaction (reducing the message history before each request) is not optional — it directly reduces your TTFT (time to first token).

**Inference Engines** (Chapter 4.3): vLLM and SGLang are the two production-grade open-source inference engines. Both expose an OpenAI-compatible API. vLLM is the most widely deployed. SGLang is faster on structured generation and multi-step reasoning tasks. For Euclid cloud deployment, you will use either vLLM or Ollama depending on your GPU setup. On a single consumer GPU (RTX 4090, A6000), use Ollama. On a multi-GPU cloud instance (A100x8, H100x8), use vLLM with tensor parallelism.

**Autoscaling and Concurrency** (Chapter 7.2): Each Ollama instance is single-concurrency by default. For a multi-user deployment, you need either: (a) multiple Ollama instances behind a load balancer, or (b) vLLM with continuous batching which handles multiple requests concurrently with the same model weights. For Euclid's initial deployment, you will run one Ollama instance with request queueing in the backend.

### 2.4 SenecaChat-master.zip — The Frontend and Server Patterns

SenecaChat is a 4,740-line single-file Express + SQLite + Ollama agentic chat system. It is the most directly applicable reference for Euclid's architecture because it solves the same problem: wrapping Ollama in an agentic runtime with a web frontend. You will use its frontend design (the `public/index.html` single-page application) and adapt its server patterns for the Euclid backend.

The key components you extract from SenecaChat are:

The **circuit breaker** class that wraps Ollama calls with 4-failure threshold, 20-second recovery timeout, and half-open probing. This is not optional — Ollama can crash, hang, or reject connections under load, and without a circuit breaker your entire server will pile up stalled requests.

The **BM25 search** implementation that indexes document chunks and retrieves the most relevant ones by query. The implementation tokenizes both query and documents, computes BM25 scores using document frequency and term frequency, applies a 3-results-per-document cap, and supports a hybrid mode that adds cosine similarity. You will use this for both the RAG (document knowledge base) and the memory search features.

The **27-table SQLite schema** (schema-v18.js) which you will adapt to add Euclid-specific tables. The important tables are: `conversations`, `messages`, `memory`, `agent_memory`, `plans`, `tasks`, `cost_events`, `heartbeat_runs`, `agent_wakeup_requests`, `activity_log_v2`, `issues`, and `workspaces`. You will keep all of these and add: `tool_calls`, `tool_results`, `skill_usage`, `file_snapshots`, and `permission_decisions`.

The **context management** thresholds: 60% → warn, 80% → compact, 85% → minimal prompt mode. These are empirically derived numbers (SenecaChat updated from 70% to 60% based on research showing model quality degrades before the nominal window limit). You will use these exact thresholds in Euclid.

The **Reflexion** implementation (`POST /api/reflect`) which runs a 5-step self-critique on failed responses: identify what went wrong, diagnose the root cause, generate a better approach, produce an improved answer. This comes directly from the Shinn et al. (2022) Reflexion paper. You will include this as a first-class feature in Euclid.

---

## 3. Tech Stack Decision

Euclid is built with the following stack. These decisions are final and must not be changed.

```
Runtime:          Node.js 20+ (LTS)
Server:           Express 4
Database:         better-sqlite3 (synchronous SQLite — no async driver confusion)
AI Client:        ollama npm package + direct fetch for streaming
Schema:           Raw SQL with prepared statements (no ORM)
Auth:             Bearer token (same as SenecaChat)
Frontend:         SenecaChat's public/index.html (adapted for Euclid)
Process manager:  PM2 (production) / nodemon (development)
```

Do not use TypeScript for the backend. The Claude Code source (src.zip) uses TypeScript because it uses Bun's type-checking and dead-code elimination at build time. Euclid runs on Node.js and building a TypeScript compilation pipeline adds friction without benefit for a single-process server. Write clean, well-documented JavaScript with JSDoc comments. If type safety is needed for tool schemas, use Zod in JavaScript mode.

Do not use Prisma, Sequelize, or any ORM. The SenecaChat database layer (`src/db/index.js`) uses raw SQL with `better-sqlite3` prepared statements and this is correct. Prepared statements are pre-compiled, prevent SQL injection, and run faster than any ORM. The database layer must be synchronous (better-sqlite3) because all operations are fast and synchronous I/O in Node.js is appropriate for SQLite on localhost.

---

## 4. Project Scaffold

The AI must create the following directory structure exactly. Every directory serves a specific purpose, derived from the Claude Code architecture.

```bash
mkdir -p euclid/{src,public,data,uploads,logs,skills,agents}
mkdir -p euclid/src/{db,utils,tools,routes,services,middleware,agents}
mkdir -p euclid/src/tools/{bash,read,write,edit,glob,grep,ls,agent,mcp,search,todo}
mkdir -p euclid/src/services/{ollama,compact,rag,skill,task,permission}
mkdir -p euclid/src/agents/{coder,researcher,writer,analyst,critic,planner}
mkdir -p euclid/skills/{api-design,backend-patterns,coding-standards,e2e-testing,frontend-patterns}
cd euclid
```

Now initialize the Node.js project:

```bash
npm init -y
npm install express better-sqlite3 ollama node-fetch@2 multer helmet \
  express-rate-limit uuid crypto zod morgan compression
npm install --save-dev nodemon
```

Create the main entry point:

```bash
touch src/server.js src/db/index.js src/db/schema.js src/utils/index.js \
  src/utils/systemPrompt.js src/utils/bm25.js src/utils/context.js \
  src/middleware/auth.js src/middleware/rateLimit.js \
  src/routes/chat.js src/routes/memory.js src/routes/docs.js \
  src/routes/conversations.js src/routes/agents.js src/routes/exec.js \
  src/routes/plans.js src/routes/evals.js src/routes/system.js \
  src/services/ollama/client.js src/services/ollama/circuitBreaker.js \
  src/services/ollama/stream.js src/services/compact/autoCompact.js \
  src/services/compact/buildPostCompact.js \
  src/services/rag/ingest.js src/services/rag/retrieve.js \
  src/services/skill/loader.js src/services/skill/selector.js \
  src/services/task/runner.js src/services/permission/checker.js \
  src/tools/index.js src/tools/bash/bash.js src/tools/read/read.js \
  src/tools/write/write.js src/tools/edit/edit.js \
  src/tools/glob/glob.js src/tools/grep/grep.js src/tools/ls/ls.js \
  src/tools/agent/agentTool.js src/tools/todo/todo.js \
  src/tools/search/webSearch.js
```

Create `package.json` scripts:

```json
{
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js",
    "db:reset": "node -e \"require('./src/db').reset()\"",
    "test": "node --test src/**/*.test.js"
  }
}
```

---

## 5. Inference Layer — Ollama on Cloud GPU

This section covers everything from Chapters 3-5 of Inference Engineering by Philip Kiely, applied to Euclid's specific needs. The AI must understand this layer before building any application code.

### 5.1 Choosing Your Deployment Mode

Euclid supports three deployment modes for the inference backend:

**Mode 1: Local Ollama (Development)**. Ollama runs on the same machine as the Euclid server. This requires a GPU with at least 8 GB VRAM for small models (7B at Q4) or 48 GB for large models (32B at Q4). On macOS with Apple Silicon, Ollama uses the unified memory architecture and can run 7B models on 16 GB RAM and 32B models on 64 GB RAM.

**Mode 2: Remote Ollama on Cloud GPU**. Ollama runs on a rented GPU instance (RunPod, Vast.ai, Lambda Labs, or Hetzner). The Euclid server calls it over the network. This is the recommended production mode for personal use. Ollama exposes port 11434. You configure `OLLAMA_BASE_URL=http://<cloud-ip>:11434` in Euclid's environment.

**Mode 3: vLLM or SGLang on Cloud GPU (Multi-user Production)**. vLLM or SGLang replaces Ollama entirely. Both expose an OpenAI-compatible API at `/v1/chat/completions`. You configure `USE_OPENAI_COMPAT=true` and `OLLAMA_BASE_URL=http://<cloud-ip>:8000` in Euclid's environment. Euclid's Ollama client automatically switches to the OpenAI format when this flag is set.

### 5.2 Cloud GPU Setup — RunPod

The AI must generate a complete setup script for deploying Ollama on RunPod. RunPod is the best cost-to-performance option for Euclid because it charges per-second, supports spot instances, and has pre-built Ollama templates.

Create the file `scripts/setup-runpod.sh`:

```bash
#!/bin/bash
# Run this script ONCE on a fresh RunPod GPU pod (any GPU with 24+ GB VRAM)
# Recommended: RTX 4090 (24 GB) for 7B-13B models, A100-80GB for 70B models

set -e

echo "Installing Ollama..."
curl -fsSL https://ollama.ai/install.sh | sh

echo "Starting Ollama service..."
OLLAMA_HOST=0.0.0.0 ollama serve &
sleep 5

echo "Pulling recommended models..."
# Primary model — best coding performance at 32B
ollama pull qwen2.5-coder:32b-instruct-q4_K_M

# Draft model for speculative decoding (optional but gives ~2x speedup)
ollama pull qwen2.5-coder:1.5b

# Fallback model for tasks that don't need full 32B quality
ollama pull qwen2.5-coder:7b-instruct-q4_K_M

# Model for summarization and compaction tasks (fast, cheap)
ollama pull qwen2.5:3b

echo "Ollama is running at http://0.0.0.0:11434"
echo "Test it with: curl http://localhost:11434/api/tags"
```

### 5.3 Model Selection for Euclid

Based on the Inference Engineering book's guidance on model selection (Chapter 1.3) and the current open model landscape, Euclid uses different models for different tasks. This is the multi-model strategy:

```javascript
// src/utils/modelRouter.js
// This function selects the optimal model for each operation.
// Never use the full 32B model for cheap operations — it wastes KV cache
// and increases latency unnecessarily.

const MODEL_ROUTES = {
  // Main agentic loop — needs strong instruction following and tool use
  main: process.env.EUCLID_MAIN_MODEL || 'qwen2.5-coder:32b-instruct-q4_K_M',

  // Compaction / summarization — fast and cheap
  compact: process.env.EUCLID_COMPACT_MODEL || 'qwen2.5:3b',

  // Tool use summary (haiku equivalent) — runs after each tool batch
  summary: process.env.EUCLID_SUMMARY_MODEL || 'qwen2.5-coder:7b-instruct-q4_K_M',

  // LLM-as-judge for eval — needs balanced reasoning
  judge: process.env.EUCLID_JUDGE_MODEL || 'qwen2.5-coder:7b-instruct-q4_K_M',

  // Reflexion / self-critique — needs strong reasoning
  reflexion: process.env.EUCLID_REFLEXION_MODEL || 'qwen2.5-coder:32b-instruct-q4_K_M',

  // Intent classification — tiny model, runs on every message
  classify: process.env.EUCLID_CLASSIFY_MODEL || 'qwen2.5:3b',
};

function getModel(role) {
  return MODEL_ROUTES[role] || MODEL_ROUTES.main;
}

module.exports = { getModel, MODEL_ROUTES };
```

### 5.4 The Ollama Client

The Ollama client is the lowest-level abstraction in Euclid. It wraps HTTP calls to Ollama, handles streaming with SSE parsing, applies the circuit breaker, and normalizes the response format. The client must support both Ollama's native `/api/chat` endpoint and the OpenAI-compatible `/v1/chat/completions` endpoint so you can swap inference backends without changing application code.

Write `src/services/ollama/client.js` with the following complete implementation:

```javascript
'use strict';

const fetch = require('node-fetch');
const { CircuitBreaker } = require('./circuitBreaker');
const { getModel } = require('../../utils/modelRouter');

const BASE_URL = process.env.OLLAMA_BASE_URL || 'http://localhost:11434';
const USE_OPENAI_COMPAT = process.env.USE_OPENAI_COMPAT === 'true';

// One circuit breaker per base URL — shared across all request types
const breaker = new CircuitBreaker('ollama', { threshold: 4, resetTimeout: 20000 });

// Maximum retry attempts with exponential backoff
const MAX_RETRIES = 3;
const BASE_DELAY = 500; // ms

async function withRetry(fn, opts = {}) {
  const { maxRetries = MAX_RETRIES, baseDelay = BASE_DELAY, shouldRetry } = opts;
  let lastErr;
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    if (attempt > 0) {
      const delay = Math.min(baseDelay * Math.pow(2, attempt - 1), 8000);
      await new Promise(r => setTimeout(r, delay + Math.random() * delay * 0.3));
    }
    try {
      return await fn(attempt);
    } catch (e) {
      lastErr = e;
      if (e.name === 'AbortError') break;
      if (shouldRetry && !shouldRetry(e, attempt)) break;
    }
  }
  throw lastErr;
}

// Build the request body in the correct format for the active backend.
// Ollama's /api/chat format and OpenAI's /v1/chat/completions format are
// almost identical but have small differences in field names.
function buildRequestBody(params) {
  const { model, messages, tools, stream = true, options = {} } = params;

  if (USE_OPENAI_COMPAT) {
    return {
      model,
      messages,
      tools: tools?.length ? tools : undefined,
      stream,
      temperature: options.temperature ?? 0.1,
      max_tokens: options.num_predict ?? 8192,
    };
  }

  // Ollama native format
  return {
    model,
    messages,
    tools: tools?.length ? tools : undefined,
    stream,
    options: {
      temperature: options.temperature ?? 0.1,
      num_predict: options.num_predict ?? 8192,
      num_ctx: options.num_ctx ?? 32768,
      ...options,
    },
  };
}

function buildEndpoint(path) {
  if (USE_OPENAI_COMPAT) {
    return `${BASE_URL}/v1/chat/completions`;
  }
  return `${BASE_URL}${path}`;
}

// Non-streaming completion — used for compaction, eval, classification.
// Returns the complete message object.
async function complete(params) {
  if (!breaker.canExecute()) {
    throw new Error(`Circuit breaker OPEN for ${BASE_URL}. Ollama is unavailable.`);
  }

  try {
    const body = buildRequestBody({ ...params, stream: false });
    const response = await withRetry(async () => {
      const res = await fetch(buildEndpoint('/api/chat'), {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(body),
      });
      if (!res.ok) {
        const errText = await res.text();
        throw new Error(`Ollama HTTP ${res.status}: ${errText}`);
      }
      return res.json();
    });

    breaker.onSuccess();

    // Normalize to a consistent response shape
    if (USE_OPENAI_COMPAT) {
      return {
        message: response.choices?.[0]?.message,
        usage: response.usage,
        model: response.model,
      };
    }
    return response;
  } catch (err) {
    breaker.onFailure();
    throw err;
  }
}

// Streaming completion — used for the main agentic loop and chat.
// Returns an async generator that yields parsed message chunks.
// Each chunk has the shape: { type, delta, message, done }
async function* stream(params, signal) {
  if (!breaker.canExecute()) {
    throw new Error(`Circuit breaker OPEN for ${BASE_URL}. Ollama is unavailable.`);
  }

  const body = buildRequestBody({ ...params, stream: true });

  let response;
  try {
    response = await fetch(buildEndpoint('/api/chat'), {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
      signal,
    });
  } catch (err) {
    breaker.onFailure();
    throw err;
  }

  if (!response.ok) {
    breaker.onFailure();
    const errText = await response.text();
    throw new Error(`Ollama HTTP ${response.status}: ${errText}`);
  }

  // Parse the NDJSON stream — each line is a complete JSON object.
  // Ollama sends: {"message":{"role":"assistant","content":"..."},"done":false}
  // The final chunk has "done": true and includes usage statistics.
  let buffer = '';
  let fullContent = '';
  let toolCalls = [];

  try {
    for await (const chunk of response.body) {
      buffer += chunk.toString();
      const lines = buffer.split('\n');
      buffer = lines.pop() || '';

      for (const line of lines) {
        const trimmed = line.trim();
        if (!trimmed) continue;

        let parsed;
        try {
          parsed = JSON.parse(trimmed);
        } catch {
          continue; // Skip malformed lines
        }

        if (USE_OPENAI_COMPAT) {
          // OpenAI SSE format: data: {"choices":[{"delta":{"content":"..."}}]}
          const delta = parsed.choices?.[0]?.delta;
          if (delta?.content) {
            fullContent += delta.content;
            yield { type: 'text_delta', delta: delta.content };
          }
          if (delta?.tool_calls) {
            toolCalls.push(...delta.tool_calls);
          }
          if (parsed.choices?.[0]?.finish_reason === 'stop' || parsed.choices?.[0]?.finish_reason === 'tool_calls') {
            breaker.onSuccess();
            yield { type: 'done', content: fullContent, toolCalls, usage: parsed.usage };
          }
        } else {
          // Ollama native format
          if (parsed.message?.content) {
            fullContent += parsed.message.content;
            yield { type: 'text_delta', delta: parsed.message.content };
          }
          if (parsed.message?.tool_calls?.length) {
            toolCalls.push(...parsed.message.tool_calls);
          }
          if (parsed.done) {
            breaker.onSuccess();
            yield {
              type: 'done',
              content: fullContent,
              toolCalls,
              usage: {
                prompt_tokens: parsed.prompt_eval_count || 0,
                completion_tokens: parsed.eval_count || 0,
              },
            };
          }
        }
      }
    }
  } catch (err) {
    if (err.name !== 'AbortError') {
      breaker.onFailure();
    }
    throw err;
  }
}

module.exports = { complete, stream, breaker, withRetry, BASE_URL };
```

---

## 6. The Core Agentic Loop

The agentic loop is the heart of Euclid. Everything else serves this loop. Understand it completely before implementing anything else. The loop in `src/query.ts` from the Claude Code source establishes the canonical pattern, adapted here for Ollama.

The loop has exactly five phases per iteration:

**Phase 1 — Prepare.** Build the message history that will be sent to the model. This involves applying compaction (if the context is too long), applying tool result budgets (truncating oversized tool outputs), and injecting queued attachments (memory summaries, skill discoveries). The history must be built fresh each iteration — it is not a simple append because compaction can rewrite arbitrary portions of it.

**Phase 2 — Call.** Send the prepared messages plus the system prompt and tool definitions to Ollama. Stream the response. Collect all text deltas into a single assistant message. Collect all tool_call blocks separately.

**Phase 3 — Check.** Determine whether the model requested any tool calls. If `toolCalls.length === 0`, the loop exits — the model produced a final response. If `toolCalls.length > 0`, continue to Phase 4.

**Phase 4 — Execute.** Run all requested tool calls. Each tool goes through: input validation → permission check → execution → result formatting. Tools marked `isConcurrencySafe` run in parallel. Others run sequentially. Each result is appended to `toolResults`.

**Phase 5 — Continue.** Append the assistant message and all tool results to `mutableMessages`. Check turn count against `maxTurns`. Increment turn counter. `continue` the while loop to Phase 1.

Write `src/services/queryLoop.js` with the complete implementation:

```javascript
'use strict';

const { stream, complete } = require('./ollama/client');
const { runTools } = require('../tools/runner');
const { buildCompactMessages } = require('./compact/autoCompact');
const { applyToolResultBudget } = require('./toolResultBudget');
const { getAttachmentMessages } = require('../utils/attachments');
const { db } = require('../db');
const { getModel } = require('../utils/modelRouter');
const { buildSystemPrompt } = require('../utils/systemPrompt');
const { calculateContextFill } = require('../utils/context');

// Maximum consecutive max_output_tokens recovery attempts before surfacing the error.
// Matches the Claude Code source: MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3
const MAX_RECOVERY_ATTEMPTS = 3;

// The recovery message injected when Ollama stops generating mid-thought due to
// num_predict limit. Copied verbatim from Claude Code query.ts — this exact
// phrasing has been tuned to prevent apology spirals in agentic models.
const MAX_OUTPUT_RECOVERY_MESSAGE =
  'Output token limit hit. Resume directly — no apology, no recap of what you were doing. ' +
  'Pick up mid-thought if that is where the cut happened. Break remaining work into smaller pieces.';

/**
 * queryLoop — the main agentic while-loop.
 *
 * Yields events to the caller (the /api/chat route handler) so they can be
 * streamed to the browser via SSE. Yields the following event types:
 *   { type: 'text_delta', delta: string }          — streaming text from model
 *   { type: 'tool_start', tool: string, id: string } — tool execution beginning
 *   { type: 'tool_result', id: string, result: any } — tool execution finished
 *   { type: 'system', message: string }             — system events (compaction, etc.)
 *   { type: 'done', usage: Object }                 — turn complete
 *   { type: 'error', error: string }                — unrecoverable error
 *
 * @param {Object} params
 * @param {Array} params.messages — full message history
 * @param {string} params.systemPrompt — assembled system prompt
 * @param {Array} params.tools — enabled tool definitions
 * @param {Object} params.toolContext — permission mode, CWD, session info
 * @param {AbortSignal} params.signal — abort signal for cancellation
 * @param {number} [params.maxTurns=50] — max tool-use iterations
 * @param {string} [params.model] — model override
 */
async function* queryLoop(params) {
  const {
    messages: initialMessages,
    systemPrompt,
    tools,
    toolContext,
    signal,
    maxTurns = 50,
    model = getModel('main'),
    sessionId,
  } = params;

  // Mutable state carried across loop iterations.
  // The loop body destructures this at the top of each iteration.
  let state = {
    messages: [...initialMessages],
    maxOutputRecoveryCount: 0,
    turnCount: 1,
    compacted: false,
  };

  const cumulativeUsage = { prompt_tokens: 0, completion_tokens: 0 };

  // eslint-disable-next-line no-constant-condition
  while (true) {
    const { messages, maxOutputRecoveryCount, turnCount } = state;

    // ── Phase 1: Prepare ──────────────────────────────────────────────────
    // Apply tool result budget — truncate oversized tool outputs so they do
    // not blow the context window. The budget is 32K characters per tool
    // result, with a 60/40 head/tail preservation on truncation.
    let messagesForQuery = applyToolResultBudget(messages);

    // Auto-compact if context is at or above 80% fill.
    const contextFill = calculateContextFill(messagesForQuery, model);
    if (contextFill >= 0.80) {
      yield { type: 'system', message: 'Context at 80% — running compaction...' };

      const compactResult = await buildCompactMessages({
        messages: messagesForQuery,
        systemPrompt,
        toolContext,
        model: getModel('compact'),
      });

      if (compactResult) {
        yield { type: 'system', message: `Compacted ${messages.length} messages → ${compactResult.messages.length}` };
        for (const msg of compactResult.systemMessages) {
          yield { type: 'system', message: msg };
        }
        state = {
          ...state,
          messages: compactResult.messages,
          compacted: true,
        };
        messagesForQuery = compactResult.messages;
      }
    }

    // ── Phase 2: Call ─────────────────────────────────────────────────────
    // Build the tools list in Ollama's format. Ollama accepts tools in the
    // OpenAI function-calling schema: an array of objects with type, function,
    // name, description, and parameters.
    const ollamaTools = buildOllamaToolList(tools, toolContext);

    // Build the effective system prompt. At 85% context fill, switch to
    // minimal mode which strips identity blocks to save tokens.
    const effectiveSystemPrompt = buildSystemPrompt({
      base: systemPrompt,
      contextFill,
      turnCount,
      sessionId,
      compacted: state.compacted,
    });

    const assistantContentParts = [];
    let toolCallBlocks = [];
    let currentDeltaBuffer = '';
    let hitOutputLimit = false;

    try {
      for await (const chunk of stream({
        model,
        messages: [
          { role: 'system', content: effectiveSystemPrompt },
          ...messagesForQuery,
        ],
        tools: ollamaTools,
        options: {
          temperature: 0.1,
          num_predict: 8192,
          num_ctx: getContextSize(model),
        },
      }, signal)) {
        if (signal?.aborted) break;

        if (chunk.type === 'text_delta') {
          currentDeltaBuffer += chunk.delta;
          yield chunk; // stream text to SSE clients
        }

        if (chunk.type === 'done') {
          cumulativeUsage.prompt_tokens += chunk.usage?.prompt_tokens || 0;
          cumulativeUsage.completion_tokens += chunk.usage?.completion_tokens || 0;

          // Detect if model hit its output limit (incomplete response).
          // Ollama doesn't return a finish_reason like OpenAI does — instead,
          // we detect it by checking if eval_count >= num_predict.
          if (chunk.usage?.completion_tokens >= 8000) {
            hitOutputLimit = true;
          }

          assistantContentParts.push(currentDeltaBuffer);
          toolCallBlocks = chunk.toolCalls || [];
          currentDeltaBuffer = '';
        }
      }
    } catch (err) {
      if (err.name === 'AbortError' || signal?.aborted) {
        yield { type: 'system', message: 'Request aborted by user.' };
        return;
      }
      yield { type: 'error', error: err.message };
      return;
    }

    if (signal?.aborted) {
      yield { type: 'system', message: 'Aborted.' };
      return;
    }

    const assistantContent = assistantContentParts.join('');
    const assistantMessage = {
      role: 'assistant',
      content: assistantContent || null,
      tool_calls: toolCallBlocks.length ? toolCallBlocks : undefined,
    };

    // ── Phase 3: Check ────────────────────────────────────────────────────
    // If model hit its output limit and produced no tool calls, inject the
    // recovery message and retry. This is the max_output_tokens recovery
    // pattern from Claude Code query.ts.
    if (hitOutputLimit && toolCallBlocks.length === 0) {
      if (maxOutputRecoveryCount >= MAX_RECOVERY_ATTEMPTS) {
        // Recovery exhausted — surface the truncated response and stop.
        yield { type: 'done', usage: cumulativeUsage };
        return;
      }

      const recoveryMessage = {
        role: 'user',
        content: MAX_OUTPUT_RECOVERY_MESSAGE,
      };

      yield { type: 'system', message: `Output limit recovery attempt ${maxOutputRecoveryCount + 1}/${MAX_RECOVERY_ATTEMPTS}` };

      state = {
        ...state,
        messages: [...messagesForQuery, assistantMessage, recoveryMessage],
        maxOutputRecoveryCount: maxOutputRecoveryCount + 1,
        turnCount,
      };
      continue;
    }

    // No tool calls — model is done. Emit final event and exit loop.
    if (toolCallBlocks.length === 0) {
      yield { type: 'done', usage: cumulativeUsage };
      return;
    }

    // ── Phase 4: Execute ──────────────────────────────────────────────────
    // Run all tool calls. Concurrent-safe tools run in parallel. Others
    // run sequentially. Each result is a ToolResult object.
    yield { type: 'system', message: `Executing ${toolCallBlocks.length} tool(s)...` };

    let toolResults;
    try {
      toolResults = await runTools(toolCallBlocks, tools, toolContext, signal, (event) => {
        // Forward tool progress events to SSE stream
        return event; // caller yields this
      });
    } catch (err) {
      yield { type: 'error', error: `Tool execution failed: ${err.message}` };
      return;
    }

    for (const result of toolResults) {
      yield { type: 'tool_result', id: result.id, toolName: result.name, result: result.output };
    }

    // ── Phase 5: Continue ─────────────────────────────────────────────────
    // Append assistant message and tool results to history. Check maxTurns.
    const toolResultMessages = toolResults.map(r => ({
      role: 'tool',
      tool_call_id: r.id,
      content: typeof r.output === 'string' ? r.output : JSON.stringify(r.output),
    }));

    const nextTurnCount = turnCount + 1;
    if (maxTurns && nextTurnCount > maxTurns) {
      yield { type: 'system', message: `Max turns (${maxTurns}) reached. Stopping.` };
      yield { type: 'done', usage: cumulativeUsage };
      return;
    }

    // Record tool calls to DB for observability
    recordToolUsage(sessionId, toolCallBlocks, toolResults);

    state = {
      ...state,
      messages: [
        ...messagesForQuery,
        assistantMessage,
        ...toolResultMessages,
      ],
      maxOutputRecoveryCount: 0,
      turnCount: nextTurnCount,
    };
  }
}

// Build the tool list in Ollama's OpenAI-compatible schema format.
// Only include tools that are enabled and pass permission pre-check.
function buildOllamaToolList(tools, toolContext) {
  return tools
    .filter(t => t.isEnabled())
    .map(t => ({
      type: 'function',
      function: {
        name: t.name,
        description: t.description,
        parameters: t.inputSchema,
      },
    }));
}

// Get the context size for a model. Default to 32K tokens.
// Larger models can handle more context but take longer to prefill.
function getContextSize(model) {
  if (model.includes('7b')) return 16384;
  if (model.includes('32b')) return 32768;
  if (model.includes('70b')) return 32768;
  return 32768;
}

function recordToolUsage(sessionId, toolCallBlocks, toolResults) {
  try {
    const { db: database } = require('../db');
    const stmt = database.prepare(`
      INSERT INTO tool_calls (id, session_id, tool_name, input, output, created_at)
      VALUES (?, ?, ?, ?, ?, ?)
    `);
    for (let i = 0; i < toolCallBlocks.length; i++) {
      const call = toolCallBlocks[i];
      const result = toolResults[i];
      stmt.run(
        call.id || require('uuid').v4(),
        sessionId,
        call.function?.name || call.name,
        JSON.stringify(call.function?.arguments || call.input),
        JSON.stringify(result?.output),
        Date.now(),
      );
    }
  } catch (_) { /* non-critical */ }
}

module.exports = { queryLoop };
```

---

## 7. The Tool System

The tool system is the most expansive part of Euclid. Every capability the model can invoke is a tool. Tools are not just shell commands — they include file read/write, web search, memory operations, sub-agent spawning, and more.

### 7.1 The Tool Interface

Based on `Tool.ts` from the Claude Code source, every Euclid tool must implement the following JavaScript interface. This is not TypeScript — it is a documentation contract that every tool file must satisfy. The AI building each tool must read this interface first and implement every required method.

```javascript
// The Tool interface — every tool in src/tools/ must implement these.
// This is a documentation contract, not enforced by TypeScript.

class ToolInterface {
  // ── Identity ──────────────────────────────────────────────────────────
  get name() { throw new Error('name is required'); }
  // Optional aliases for backward compat when tools are renamed
  get aliases() { return []; }
  // Short capability phrase for keyword search (3-10 words)
  get searchHint() { return ''; }

  // ── Schema ────────────────────────────────────────────────────────────
  // JSON Schema object for the tool's input parameters.
  // This is what gets passed to Ollama in the tools array.
  get inputSchema() { throw new Error('inputSchema is required'); }

  // ── Behavior flags ────────────────────────────────────────────────────
  // Returns true if the tool can run in parallel with other tools.
  // Default: false (assume not parallel-safe — fail closed)
  isConcurrencySafe(input) { return false; }

  // Returns true if the tool only reads, never writes.
  // Read-only tools get auto-approved in certain permission modes.
  isReadOnly(input) { return false; }

  // Returns true if the tool performs irreversible operations.
  // Destructive tools always prompt the user for confirmation.
  isDestructive(input) { return false; }

  // Returns true if the tool is available in the current environment.
  isEnabled() { return true; }

  // ── Execution ─────────────────────────────────────────────────────────
  // Validate the input before running. Called before checkPermissions.
  // Return { valid: true } or { valid: false, message: string, errorCode: number }
  async validateInput(input, context) { return { valid: true }; }

  // Check whether the user's permission settings allow this tool use.
  // Return { behavior: 'allow' } or { behavior: 'ask', message: string }
  // or { behavior: 'deny', message: string }
  async checkPermissions(input, context) { return { behavior: 'allow' }; }

  // Execute the tool. Return { output: any, newMessages: [...] }
  // output is what gets sent back to the model as the tool result.
  // newMessages is optional — extra messages to inject into history.
  async call(input, context, onProgress) { throw new Error('call() is required'); }

  // ── Display ───────────────────────────────────────────────────────────
  // Short human-readable name shown in the UI during tool execution.
  userFacingName(input) { return this.name; }

  // Format the tool result for the model's context.
  // This is what the model reads — keep it dense and informative.
  formatResult(output, toolUseId) {
    if (typeof output === 'string') return output;
    return JSON.stringify(output, null, 2);
  }
}
```

### 7.2 The Tool Runner

The tool runner is responsible for: receiving the list of tool call blocks from the model, finding the matching tool implementation for each, running validateInput then checkPermissions then call, and returning structured results. Tools marked `isConcurrencySafe` run in parallel via `Promise.all`. Others run sequentially.

Write `src/tools/runner.js`:

```javascript
'use strict';

const { loadTools } = require('./index');

/**
 * runTools — executes all tool calls from a model response.
 *
 * @param {Array} toolCallBlocks — tool_calls from the model response
 * @param {Array} tools — registered tool instances
 * @param {Object} context — toolContext (cwd, permissionMode, sessionId, etc.)
 * @param {AbortSignal} signal
 * @param {Function} onProgress — callback for progress events
 * @returns {Promise<Array>} — array of { id, name, output, error }
 */
async function runTools(toolCallBlocks, tools, context, signal, onProgress) {
  const results = [];

  // Separate concurrent-safe and sequential tools.
  // Run sequential tools first in order, then concurrent tools in parallel.
  // This matches the Claude Code behavior where sequential tools run first
  // because they may modify state that concurrent tools depend on.
  const sequential = [];
  const concurrent = [];

  for (const block of toolCallBlocks) {
    const tool = findTool(tools, block.function?.name || block.name);
    if (!tool) {
      results.push({
        id: block.id,
        name: block.function?.name || block.name,
        output: `Tool not found: ${block.function?.name || block.name}`,
        error: true,
      });
      continue;
    }

    const entry = { block, tool };
    if (tool.isConcurrencySafe(parseInput(block))) {
      concurrent.push(entry);
    } else {
      sequential.push(entry);
    }
  }

  // Run sequential tools in order
  for (const { block, tool } of sequential) {
    if (signal?.aborted) break;
    const result = await executeTool(tool, block, context, signal, onProgress);
    results.push(result);
  }

  // Run concurrent tools in parallel
  if (!signal?.aborted && concurrent.length > 0) {
    const concurrentResults = await Promise.all(
      concurrent.map(({ block, tool }) =>
        executeTool(tool, block, context, signal, onProgress)
      )
    );
    results.push(...concurrentResults);
  }

  return results;
}

async function executeTool(tool, block, context, signal, onProgress) {
  const input = parseInput(block);
  const toolId = block.id || require('uuid').v4();
  const toolName = tool.name;

  try {
    // Step 1: Validate input schema
    const validation = await tool.validateInput(input, context);
    if (!validation.valid) {
      return {
        id: toolId,
        name: toolName,
        output: `Input validation failed (code ${validation.errorCode}): ${validation.message}`,
        error: true,
      };
    }

    // Step 2: Check permissions
    const permission = await tool.checkPermissions(input, context);
    if (permission.behavior === 'deny') {
      return {
        id: toolId,
        name: toolName,
        output: `Permission denied: ${permission.message}`,
        error: true,
      };
    }

    if (permission.behavior === 'ask') {
      // In headless mode, auto-deny asks that can't be answered.
      // In interactive mode, this would show a UI prompt.
      if (context.isNonInteractive) {
        return {
          id: toolId,
          name: toolName,
          output: `Permission required (non-interactive mode): ${permission.message}`,
          error: true,
        };
      }
      // TODO: Interactive permission UI via SSE back-channel
    }

    // Step 3: Execute
    if (signal?.aborted) {
      return { id: toolId, name: toolName, output: 'Aborted by user.', error: true };
    }

    const result = await tool.call(input, context, onProgress);

    return {
      id: toolId,
      name: toolName,
      output: result.output,
      newMessages: result.newMessages,
      error: false,
    };

  } catch (err) {
    return {
      id: toolId,
      name: toolName,
      output: `Tool error: ${err.message}`,
      error: true,
    };
  }
}

function findTool(tools, name) {
  return tools.find(t => t.name === name || (t.aliases || []).includes(name));
}

function parseInput(block) {
  // Ollama sends tool arguments as a JSON string inside function.arguments
  // OpenAI compat sends it the same way
  const args = block.function?.arguments || block.input || '{}';
  try {
    return typeof args === 'string' ? JSON.parse(args) : args;
  } catch {
    return {};
  }
}

module.exports = { runTools };
```

### 7.3 Core Tool Implementations

Every tool below must be implemented completely. The AI must write each one as a separate file following the Tool interface above.

**Bash Tool** (`src/tools/bash/bash.js`). Executes shell commands with a configurable timeout (default 120s), a 32K character output cap with 60/40 head/tail truncation on overflow, loop detection (same command ≥4 times in 2 minutes is blocked), safety scoring (0-100 risk scale), working directory enforcement, and audit logging of every command. The tool is NOT `isConcurrencySafe` because shell commands can interfere with each other's file system state. Implement a `safetyScore(command)` function that detects: destructive flags (`rm -rf`, `dd`, `mkfs`), network write patterns (`curl | bash`, `wget | sh`), system directory writes (`/etc/`, `/sys/`, `/proc/`), eval and base64 decode chains, and recursive deletion patterns. Commands scoring over 70 require explicit user approval.

```javascript
// src/tools/bash/bash.js
'use strict';

const { execSync, exec } = require('child_process');
const crypto = require('crypto');
const path = require('path');

// Tool result size cap: 32K characters. On overflow, preserve 60% head + 40% tail.
const MAX_OUTPUT_CHARS = 32 * 1024;
const HEAD_FRACTION = 0.6;

// Loop detection: same command 4+ times in 2 minutes → blocked
const LOOP_WINDOW_MS = 2 * 60 * 1000;
const LOOP_THRESHOLD = 4;
const commandHistory = new Map(); // sessionId:hash → [timestamps]

class BashTool {
  get name() { return 'Bash'; }
  get searchHint() { return 'run shell command terminal bash script'; }

  get inputSchema() {
    return {
      type: 'object',
      required: ['command'],
      properties: {
        command: {
          type: 'string',
          description: 'The bash command to execute. Use absolute paths. Avoid interactive commands.',
        },
        timeout: {
          type: 'integer',
          description: 'Timeout in seconds (default: 120, max: 600)',
          default: 120,
        },
        description: {
          type: 'string',
          description: 'One-sentence description of what this command does and why.',
        },
      },
    };
  }

  isConcurrencySafe() { return false; }
  isReadOnly(input) {
    // Heuristic: commands that only read are safe
    const cmd = input.command?.trim() || '';
    const readOnlyPrefixes = ['cat ', 'ls ', 'echo ', 'pwd', 'which ', 'head ', 'tail ', 'grep ',
      'find ', 'wc ', 'stat ', 'file ', 'diff ', 'git log', 'git status', 'git diff', 'ps ', 'df ', 'du '];
    return readOnlyPrefixes.some(p => cmd.startsWith(p));
  }
  isDestructive(input) {
    const cmd = input.command?.trim() || '';
    return /\brm\b.*-[rf]|rmdir|dd\b|mkfs\b|shred\b/.test(cmd);
  }
  isEnabled() { return true; }

  async validateInput(input) {
    if (!input.command || typeof input.command !== 'string') {
      return { valid: false, message: 'command is required and must be a string', errorCode: 400 };
    }
    if (input.command.length > 10000) {
      return { valid: false, message: 'command is too long (max 10000 characters)', errorCode: 400 };
    }
    const timeout = input.timeout || 120;
    if (timeout > 600) {
      return { valid: false, message: 'timeout cannot exceed 600 seconds', errorCode: 400 };
    }
    return { valid: true };
  }

  async checkPermissions(input, context) {
    const score = safetyScore(input.command);
    if (score >= 90) {
      return { behavior: 'deny', message: `Safety score ${score}/100. Command blocked: ${getTopRiskReason(input.command)}` };
    }
    if (score >= 70 && context.permissionMode !== 'bypass') {
      return {
        behavior: 'ask',
        message: `Safety score ${score}/100. Reason: ${getTopRiskReason(input.command)}. Run anyway?`,
      };
    }
    return { behavior: 'allow' };
  }

  async call(input, context, onProgress) {
    const { command, timeout = 120, description = '' } = input;
    const cwd = context.cwd || process.cwd();
    const sessionId = context.sessionId || 'default';

    // Loop detection
    const cmdHash = crypto.createHash('md5').update(command.trim()).digest('hex');
    const historyKey = `${sessionId}:${cmdHash}`;
    const now = Date.now();
    const history = (commandHistory.get(historyKey) || []).filter(t => now - t < LOOP_WINDOW_MS);
    history.push(now);
    commandHistory.set(historyKey, history);

    if (history.length >= LOOP_THRESHOLD) {
      return {
        output: `[LOOP DETECTED] This exact command has been run ${history.length} times in the past 2 minutes. ` +
                `Stop repeating it. Diagnose why it is not working instead.`,
      };
    }

    // Execute with timeout
    return new Promise((resolve) => {
      let output = '';
      let stderr = '';
      let timedOut = false;

      const child = exec(command, {
        cwd,
        timeout: timeout * 1000,
        maxBuffer: MAX_OUTPUT_CHARS * 2,
        env: { ...process.env, TERM: 'dumb' },
      });

      child.stdout?.on('data', d => { output += d; });
      child.stderr?.on('data', d => { stderr += d; });

      const timeoutHandle = setTimeout(() => {
        timedOut = true;
        child.kill('SIGTERM');
      }, timeout * 1000);

      child.on('close', (code) => {
        clearTimeout(timeoutHandle);
        let combined = output + (stderr ? `\n[stderr]\n${stderr}` : '');
        let truncated = false;

        if (combined.length > MAX_OUTPUT_CHARS) {
          const headEnd = Math.floor(MAX_OUTPUT_CHARS * HEAD_FRACTION);
          const tailStart = combined.length - Math.floor(MAX_OUTPUT_CHARS * (1 - HEAD_FRACTION));
          combined = combined.slice(0, headEnd) +
            `\n\n... [OUTPUT TRUNCATED: ${combined.length - MAX_OUTPUT_CHARS} characters removed] ...\n\n` +
            combined.slice(tailStart);
          truncated = true;
        }

        const prefix = timedOut
          ? `[TIMEOUT after ${timeout}s]\n`
          : code !== 0 ? `[Exit code: ${code}]\n` : '';

        resolve({ output: prefix + combined });
      });
    });
  }

  userFacingName(input) {
    return input?.command?.slice(0, 60) || 'bash';
  }
}

// Safety scoring — returns 0-100 risk score for a shell command.
// Higher = more dangerous.
function safetyScore(cmd) {
  let score = 0;
  const c = cmd.toLowerCase();

  // Destructive file operations
  if (/\brm\s+.*-[rf]/.test(c)) score += 50;
  if (/\bdd\b/.test(c)) score += 60;
  if (/\bmkfs\b/.test(c)) score += 70;
  if (/\bshred\b/.test(c)) score += 50;
  if (/\btruncate\b/.test(c)) score += 30;

  // Network + shell execution chains (the classic supply chain attack)
  if (/curl.*\|.*sh/.test(c) || /wget.*\|.*sh/.test(c)) score += 90;
  if (/curl.*\|.*bash/.test(c) || /wget.*\|.*bash/.test(c)) score += 90;

  // Base64 decode chains
  if (/base64.*-d.*\|/.test(c) || /base64.*--decode.*\|/.test(c)) score += 80;

  // Eval with dynamic content
  if (/\beval\b/.test(c) && /\$\(/.test(c)) score += 70;

  // System directory writes
  if (/>(\/etc|\/sys|\/proc|\/boot)/.test(c)) score += 80;

  // Recursive deletion
  if (/rm.*-rf\s+\//.test(c) || /rm.*-rf\s+~/.test(c)) score += 95;

  // sudo with dangerous commands
  if (/sudo\s+(rm|dd|mkfs|shred)/.test(c)) score += 40;

  // Fork bomb pattern
  if (/:\(\)\s*\{.*\}\s*;/.test(c)) score += 100;

  return Math.min(score, 100);
}

function getTopRiskReason(cmd) {
  const c = cmd.toLowerCase();
  if (/:\(\)\s*\{.*\}\s*;/.test(c)) return 'fork bomb pattern';
  if (/curl.*\|.*sh/.test(c) || /wget.*\|.*sh/.test(c)) return 'network pipe-to-shell';
  if (/rm.*-rf\s+\//.test(c)) return 'recursive deletion of root';
  if (/\bdd\b/.test(c)) return 'disk data destruction';
  if (/\bmkfs\b/.test(c)) return 'filesystem formatting';
  if (/base64.*-d.*\|/.test(c)) return 'base64 decode chain';
  if (/\beval\b/.test(c)) return 'dynamic eval';
  return 'multiple risk factors';
}

module.exports = new BashTool();
```

**Read Tool** (`src/tools/read/read.js`). Reads a file from disk. Enforces a file size limit (1 MB by default). Supports line range reading (`start_line`, `end_line`) which is essential for large files. Returns the file content with line numbers prepended (the model needs these to make targeted edits). Marks the file in `context.readFileState` so duplicate reads can be detected. The tool is `isConcurrencySafe` and `isReadOnly`.

**Write Tool** (`src/tools/write/write.js`). Creates a new file or completely overwrites an existing one. Always shows a diff before writing (in interactive mode). Validates that the parent directory exists. Updates `context.readFileState` after writing. NOT `isConcurrencySafe`.

**Edit Tool** (`src/tools/edit/edit.js`). Replaces an exact string in a file with a new string. The `old_str` must appear exactly once in the file — if it appears zero times or multiple times, the tool returns an error. This is the surgical edit tool that Claude Code uses for most code changes. Shows the resulting diff after editing.

**Glob Tool** (`src/tools/glob/glob.js`). Lists files matching a glob pattern (e.g., `**/*.ts`, `src/**/*.js`). Returns a sorted list of matching paths. Uses the `fast-glob` npm package. Respects `.gitignore` by default.

**Grep Tool** (`src/tools/grep/grep.js`). Searches file contents for a pattern using `ripgrep` if available, otherwise falls back to Node.js `readline` line-by-line search. Returns matching lines with file paths and line numbers. Supports regex patterns.

**LS Tool** (`src/tools/ls/ls.js`). Lists directory contents with file sizes and modification times. Respects `.gitignore`. Returns a tree-formatted string. Is `isConcurrencySafe` and `isReadOnly`.

**Todo Tool** (`src/tools/todo/todo.js`). Creates, reads, and updates a per-session markdown todo checklist. Stores todos in the database. The format is Manus-style: `[ ]` pending, `[~]` in-progress, `[x]` done. The model uses this to track multi-step tasks across long sessions.

---

## 8. Session State and QueryEngine

The `QueryEngine` class from `QueryEngine.ts` owns the lifecycle of a single conversation. It holds all the mutable state that persists across turns: the message array, the running token count, the file state cache, and the abort controller. The API creates one `QueryEngine` per session and calls `submitMessage()` each time the user sends a message.

Write `src/services/queryEngine.js`:

```javascript
'use strict';

const { v4: uuid } = require('uuid');
const { queryLoop } = require('./queryLoop');
const { loadTools } = require('../tools');
const { buildSystemPrompt } = require('../utils/systemPrompt');
const { cloneFileStateCache, createFileStateCache } = require('../utils/fileStateCache');
const { db } = require('../db');
const { getModel } = require('../utils/modelRouter');

class QueryEngine {
  constructor(config) {
    const {
      cwd = process.cwd(),
      tools,
      sessionId,
      customSystemPrompt,
      appendSystemPrompt,
      permissionMode = 'default',
      maxTurns = 50,
      model,
      initialMessages = [],
    } = config;

    this.sessionId = sessionId || uuid();
    this.cwd = cwd;
    this.tools = tools || loadTools();
    this.permissionMode = permissionMode;
    this.maxTurns = maxTurns;
    this.model = model || getModel('main');
    this.customSystemPrompt = customSystemPrompt;
    this.appendSystemPrompt = appendSystemPrompt;

    // Mutable conversation state — persists across all turns in this session
    this.mutableMessages = [...initialMessages];
    this.readFileState = createFileStateCache();
    this.abortController = new AbortController();

    // Cumulative usage across all turns
    this.totalUsage = { prompt_tokens: 0, completion_tokens: 0 };
  }

  // Build the ToolUseContext passed to every tool during a turn.
  // This is a snapshot of the current session state plus the tools.
  buildToolContext() {
    return {
      cwd: this.cwd,
      sessionId: this.sessionId,
      permissionMode: this.permissionMode,
      readFileState: this.readFileState,
      isNonInteractive: false,
      tools: this.tools,
    };
  }

  // submitMessage is called once per user turn.
  // It appends the user message, runs the query loop, and returns an async
  // generator of events that the route handler streams to the SSE client.
  async* submitMessage(userMessage, opts = {}) {
    // Append user message to history
    this.mutableMessages.push({
      role: 'user',
      content: typeof userMessage === 'string' ? userMessage : JSON.stringify(userMessage),
    });

    // Assemble the system prompt for this turn
    const systemPrompt = buildSystemPrompt({
      custom: this.customSystemPrompt,
      append: this.appendSystemPrompt,
      cwd: this.cwd,
      sessionId: this.sessionId,
    });

    // Reset abort controller for this turn (previous turn's signal is stale)
    this.abortController = new AbortController();
    const { signal } = this.abortController;

    // Run the agentic loop
    const loopEvents = queryLoop({
      messages: this.mutableMessages,
      systemPrompt,
      tools: this.tools,
      toolContext: this.buildToolContext(),
      signal,
      maxTurns: opts.maxTurns || this.maxTurns,
      model: opts.model || this.model,
      sessionId: this.sessionId,
    });

    let lastAssistantContent = '';
    const newMessages = [];

    for await (const event of loopEvents) {
      yield event;

      // Accumulate the final assistant message for history
      if (event.type === 'text_delta') {
        lastAssistantContent += event.delta;
      }
      if (event.type === 'tool_result') {
        newMessages.push({
          role: 'tool',
          tool_call_id: event.id,
          content: typeof event.result === 'string' ? event.result : JSON.stringify(event.result),
        });
      }
      if (event.type === 'done') {
        this.totalUsage.prompt_tokens += event.usage?.prompt_tokens || 0;
        this.totalUsage.completion_tokens += event.usage?.completion_tokens || 0;
      }
    }

    // Append the final assistant message to history
    if (lastAssistantContent) {
      this.mutableMessages.push({ role: 'assistant', content: lastAssistantContent });
    }
    this.mutableMessages.push(...newMessages);

    // Persist the updated conversation to DB
    this._persistToDb();
  }

  // Abort the current turn immediately
  abort(reason = 'user') {
    this.abortController.abort(reason);
  }

  // Restore a session from persisted DB state (used by /resume)
  static fromDatabase(sessionId) {
    const conv = db.prepare('SELECT * FROM conversations WHERE id = ?').get(sessionId);
    if (!conv) return null;

    let messages = [];
    try { messages = JSON.parse(conv.messages || '[]'); } catch { messages = []; }

    return new QueryEngine({
      sessionId,
      initialMessages: messages,
      model: conv.model,
      permissionMode: conv.permission_mode || 'default',
    });
  }

  _persistToDb() {
    try {
      db.prepare(`
        INSERT INTO conversations (id, messages, updated_at)
        VALUES (?, ?, ?)
        ON CONFLICT(id) DO UPDATE SET messages = excluded.messages, updated_at = excluded.updated_at
      `).run(
        this.sessionId,
        JSON.stringify(this.mutableMessages),
        Date.now(),
      );
    } catch (err) {
      console.error('Failed to persist conversation:', err.message);
    }
  }
}

module.exports = { QueryEngine };
```

---

## 9. Database Schema

The database schema is the ground truth for everything Euclid persists. It is derived from SenecaChat's `schema-v18.js` (27 tables) with Euclid-specific additions. The AI must create `src/db/schema.js` with the complete schema and `src/db/index.js` with all database operations.

The schema must be created as a single SQL string executed with `db.exec()` on initialization. Every table uses `IF NOT EXISTS` so the schema is idempotent on restart.

Write `src/db/schema.js` with all of the following tables and their exact column definitions:

```javascript
'use strict';

const SCHEMA = `
  PRAGMA journal_mode = WAL;
  PRAGMA synchronous = NORMAL;
  PRAGMA foreign_keys = ON;
  PRAGMA cache_size = -64000;

  -- ── Conversations & Messages ──────────────────────────────────────────
  CREATE TABLE IF NOT EXISTS conversations (
    id TEXT PRIMARY KEY,
    name TEXT DEFAULT '',
    model TEXT DEFAULT '',
    permission_mode TEXT DEFAULT 'default',
    messages TEXT DEFAULT '[]',
    total_input_tokens INTEGER DEFAULT 0,
    total_output_tokens INTEGER DEFAULT 0,
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000),
    updated_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );
  CREATE INDEX IF NOT EXISTS idx_conv_updated ON conversations(updated_at DESC);

  -- ── Tool Calls (Euclid-specific observability) ─────────────────────
  CREATE TABLE IF NOT EXISTS tool_calls (
    id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    tool_name TEXT NOT NULL,
    input TEXT DEFAULT '{}',
    output TEXT DEFAULT '',
    duration_ms INTEGER DEFAULT 0,
    error INTEGER DEFAULT 0,
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );
  CREATE INDEX IF NOT EXISTS idx_tool_calls_session ON tool_calls(session_id, created_at DESC);
  CREATE INDEX IF NOT EXISTS idx_tool_calls_name ON tool_calls(tool_name);

  -- ── Memory (namespaced K/V store) ──────────────────────────────────
  CREATE TABLE IF NOT EXISTS memory (
    id TEXT PRIMARY KEY,
    namespace TEXT NOT NULL DEFAULT 'project_facts',
    key TEXT NOT NULL,
    content TEXT NOT NULL,
    confidence REAL DEFAULT 0.8,
    access_count INTEGER DEFAULT 0,
    last_accessed INTEGER,
    expires_at INTEGER,
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000),
    updated_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000),
    UNIQUE(namespace, key)
  );
  CREATE INDEX IF NOT EXISTS idx_memory_namespace ON memory(namespace, updated_at DESC);
  CREATE INDEX IF NOT EXISTS idx_memory_confidence ON memory(confidence DESC);

  -- ── Agent K/V Scratchpad ────────────────────────────────────────────
  CREATE TABLE IF NOT EXISTS agent_memory (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL,
    updated_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );

  -- ── Documents & RAG ────────────────────────────────────────────────
  CREATE TABLE IF NOT EXISTS docs (
    id TEXT PRIMARY KEY,
    filename TEXT NOT NULL,
    content_hash TEXT NOT NULL UNIQUE,
    size_bytes INTEGER DEFAULT 0,
    chunk_count INTEGER DEFAULT 0,
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );

  CREATE TABLE IF NOT EXISTS doc_chunks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    doc_id TEXT NOT NULL REFERENCES docs(id) ON DELETE CASCADE,
    filename TEXT NOT NULL,
    rel_path TEXT DEFAULT '',
    text TEXT NOT NULL,
    len INTEGER NOT NULL DEFAULT 0,
    freq TEXT DEFAULT '{}',
    chunk_index INTEGER DEFAULT 0,
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );
  CREATE INDEX IF NOT EXISTS idx_chunks_doc ON doc_chunks(doc_id);

  -- ── File Snapshots (for diff / rollback) ────────────────────────────
  CREATE TABLE IF NOT EXISTS file_snapshots (
    id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    file_path TEXT NOT NULL,
    content_before TEXT,
    content_after TEXT,
    tool_call_id TEXT,
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );
  CREATE INDEX IF NOT EXISTS idx_snapshots_session ON file_snapshots(session_id, created_at DESC);
  CREATE INDEX IF NOT EXISTS idx_snapshots_path ON file_snapshots(file_path);

  -- ── Permission Decisions (audit trail) ──────────────────────────────
  CREATE TABLE IF NOT EXISTS permission_decisions (
    id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    tool_name TEXT NOT NULL,
    input_summary TEXT DEFAULT '',
    decision TEXT NOT NULL,
    risk_score INTEGER DEFAULT 0,
    decided_by TEXT DEFAULT 'system',
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );
  CREATE INDEX IF NOT EXISTS idx_perm_session ON permission_decisions(session_id, created_at DESC);

  -- ── Tasks (background work items) ──────────────────────────────────
  CREATE TABLE IF NOT EXISTS tasks (
    id TEXT PRIMARY KEY,
    session_id TEXT DEFAULT '',
    type TEXT NOT NULL DEFAULT 'local_bash',
    status TEXT NOT NULL DEFAULT 'pending',
    description TEXT NOT NULL,
    output_path TEXT DEFAULT '',
    output_offset INTEGER DEFAULT 0,
    error TEXT DEFAULT '',
    started_at INTEGER,
    ended_at INTEGER,
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000),
    updated_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );
  CREATE INDEX IF NOT EXISTS idx_tasks_status ON tasks(status, created_at DESC);
  CREATE INDEX IF NOT EXISTS idx_tasks_session ON tasks(session_id);

  -- ── Plans ──────────────────────────────────────────────────────────
  CREATE TABLE IF NOT EXISTS plans (
    id TEXT PRIMARY KEY,
    session_id TEXT DEFAULT '',
    task TEXT NOT NULL,
    steps TEXT DEFAULT '[]',
    status TEXT DEFAULT 'active',
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000),
    updated_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );

  -- ── Todos (per-session checklist) ───────────────────────────────────
  CREATE TABLE IF NOT EXISTS todos (
    id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    content TEXT NOT NULL DEFAULT '',
    items TEXT DEFAULT '[]',
    updated_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );
  CREATE UNIQUE INDEX IF NOT EXISTS idx_todos_session ON todos(session_id);

  -- ── Notes ──────────────────────────────────────────────────────────
  CREATE TABLE IF NOT EXISTS notes (
    id TEXT PRIMARY KEY,
    content TEXT NOT NULL,
    tag TEXT DEFAULT '',
    pinned INTEGER DEFAULT 0,
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000),
    updated_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );

  -- ── Templates ──────────────────────────────────────────────────────
  CREATE TABLE IF NOT EXISTS templates (
    id TEXT PRIMARY KEY,
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    category TEXT DEFAULT 'general',
    use_count INTEGER DEFAULT 0,
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );

  -- ── Skill Usage Tracking (Euclid-specific) ──────────────────────────
  CREATE TABLE IF NOT EXISTS skill_usage (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL,
    skill_name TEXT NOT NULL,
    activated_by TEXT DEFAULT 'keyword',
    turn_count INTEGER DEFAULT 0,
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );
  CREATE INDEX IF NOT EXISTS idx_skill_session ON skill_usage(session_id);

  -- ── Cost & Token Tracking ───────────────────────────────────────────
  CREATE TABLE IF NOT EXISTS cost_events (
    id TEXT PRIMARY KEY,
    session_id TEXT DEFAULT '',
    model TEXT NOT NULL DEFAULT '',
    provider TEXT DEFAULT 'ollama',
    input_tokens INTEGER DEFAULT 0,
    output_tokens INTEGER DEFAULT 0,
    cost_cents INTEGER DEFAULT 0,
    occurred_at INTEGER NOT NULL,
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );
  CREATE INDEX IF NOT EXISTS idx_cost_session ON cost_events(session_id);
  CREATE INDEX IF NOT EXISTS idx_cost_occurred ON cost_events(occurred_at DESC);

  -- ── Metrics ────────────────────────────────────────────────────────
  CREATE TABLE IF NOT EXISTS metrics (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    type TEXT NOT NULL,
    value REAL NOT NULL,
    meta TEXT DEFAULT '{}',
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );
  CREATE INDEX IF NOT EXISTS idx_metrics_type ON metrics(type, created_at DESC);

  -- ── Eval Suites ─────────────────────────────────────────────────────
  CREATE TABLE IF NOT EXISTS eval_suites (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    model TEXT DEFAULT '',
    tests TEXT DEFAULT '[]',
    last_run_at INTEGER,
    last_results TEXT DEFAULT '{}',
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );

  -- ── Feedback ───────────────────────────────────────────────────────
  CREATE TABLE IF NOT EXISTS feedback (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    message_id TEXT NOT NULL,
    session_id TEXT DEFAULT '',
    rating TEXT NOT NULL CHECK(rating IN ('up','down')),
    comment TEXT DEFAULT '',
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );

  -- ── Audit Log ──────────────────────────────────────────────────────
  CREATE TABLE IF NOT EXISTS audit_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    action TEXT NOT NULL,
    details TEXT DEFAULT '{}',
    session_id TEXT DEFAULT '',
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );
  CREATE INDEX IF NOT EXISTS idx_audit_action ON audit_log(action, created_at DESC);

  -- ── Secrets (API keys stored encrypted) ─────────────────────────────
  CREATE TABLE IF NOT EXISTS secrets (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL,
    updated_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );

  -- ── Prefs ──────────────────────────────────────────────────────────
  CREATE TABLE IF NOT EXISTS prefs (
    id INTEGER PRIMARY KEY CHECK (id = 1),
    data TEXT DEFAULT '{}',
    updated_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );

  -- ── Request Traces ─────────────────────────────────────────────────
  CREATE TABLE IF NOT EXISTS request_traces (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    method TEXT NOT NULL,
    path TEXT NOT NULL,
    status_code INTEGER NOT NULL,
    latency_ms INTEGER NOT NULL,
    ip TEXT DEFAULT '',
    user_agent TEXT DEFAULT '',
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );
  CREATE INDEX IF NOT EXISTS idx_traces_path ON request_traces(path, created_at DESC);

  -- ── Agents (multi-agent registry) ──────────────────────────────────
  CREATE TABLE IF NOT EXISTS agents (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL UNIQUE,
    type TEXT DEFAULT 'custom',
    model TEXT DEFAULT '',
    system_prompt TEXT DEFAULT '',
    temperature REAL DEFAULT 0.3,
    enabled INTEGER DEFAULT 1,
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000),
    updated_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );

  -- ── Errors Log ─────────────────────────────────────────────────────
  CREATE TABLE IF NOT EXISTS errors_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    context TEXT NOT NULL,
    message TEXT NOT NULL,
    stack TEXT DEFAULT '',
    session_id TEXT DEFAULT '',
    created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
  );
`;

module.exports = { SCHEMA };
```

Write `src/db/index.js` as the database access layer. It must use `better-sqlite3`, enable WAL mode on startup, run the schema migration, and export prepared-statement based functions for every table. Every write must use a prepared statement. Never use string interpolation in SQL.

---

## 10. The API Server

Write `src/server.js` as the Express application entry point. It wires together all route handlers, middleware, and the database initialization. The server must start in under 500ms (lazy-load heavy modules). All routes are under `/api/`. The static frontend is served from `public/index.html`.

The server must implement these route groups, each in their own file under `src/routes/`:

**`/api/chat`** — The primary endpoint. Accepts `{ model, messages, sessionId, maxTurns, permissionMode }`. Creates or resumes a `QueryEngine` for the session. Calls `engine.submitMessage()`. Streams the yielded events to the client via SSE (`Content-Type: text/event-stream`). Accepts `POST /api/abort/:sessionId` to cancel.

**`/api/exec`** — Unified shell execution endpoint adapted from SenecaChat. Accepts `{ command, sessionId, cwd, bypassCache, stream }`. Safety scoring is applied before execution. High-risk commands (score ≥70) require explicit `{ force: true }` flag or are rejected. Results are cached by command hash for 60 seconds (except when `bypassCache: true`).

**`/api/conversations`** — Full CRUD for conversations. Lists, gets, creates, updates, deletes, and bulk-deletes conversations. Exports conversations as Markdown, JSON, or HTML. Full-text search via `searchConversations(query)`.

**`/api/memory`** — CRUD for the namespaced memory store. Supports `?namespace=` and `?q=` query parameters. The `GET /api/memory/session-summary` endpoint synthesizes a continuity prompt from key facts, recent episodes, and known errors — this is injected into the system prompt on the first turn of a resumed session.

**`/api/docs`** — Document ingestion and RAG. `POST /api/docs/ingest` accepts base64-encoded documents, extracts text, chunks it at 1000 characters with 200-character overlap, builds the BM25 index. `POST /api/docs/search` runs BM25 retrieval.

**`/api/agents`** — Multi-agent orchestration. `POST /api/agents/orchestrate` decomposes a task into subtasks and runs each on a specialist agent. `POST /api/agents/debate` runs for/against debate. `POST /api/agents/peer-review` scores content on 5 dimensions.

**`/api/reflect`** — Reflexion endpoint. Accepts `{ model, query, response, error }`. Runs a 5-step self-critique and returns an improved response.

**`/api/eval`** — LLM-as-judge scoring and regression test suite management.

**`/api/system`** — Health check, metrics, audit log, error log, and live log SSE stream.

---

## 11. Context Management and Compaction

Context management is the difference between a useful agent and one that hallucates or stops working after 20 turns. The Claude Code source handles this through a multi-layered system: tool result budgets, microcompact, snip compaction, and full autocompact. Euclid implements three of these four layers, omitting microcompact (which is an internal Anthropic optimization for their API's prompt cache that does not apply to Ollama).

### 11.1 Token Counting

Write `src/utils/context.js` with an accurate token count estimator. Ollama does not expose a tokenization endpoint, so you estimate using character count divided by 3.5 (a conservative estimate that works across most tokenizers). For a more accurate estimate, call `POST /api/tokenize` on the Ollama API.

```javascript
'use strict';

// Approximate tokens from character count.
// The 3.5 divisor is calibrated for English text with code.
// It is deliberately conservative (underestimates) so the compaction
// threshold is not triggered too early.
function estimateTokens(text) {
  if (!text || typeof text !== 'string') return 0;
  return Math.ceil(text.length / 3.5);
}

// Estimate total tokens for an array of messages.
function estimateMessagesTokens(messages) {
  let total = 0;
  for (const msg of messages) {
    const content = typeof msg.content === 'string' ? msg.content : JSON.stringify(msg.content);
    total += estimateTokens(content);
    total += 4; // per-message overhead (role, separators)
  }
  return total;
}

// Get the context window size for a model.
function getContextWindow(model) {
  if (!model) return 32768;
  if (model.includes('3b') || model.includes('1.5b')) return 8192;
  if (model.includes('7b')) return 16384;
  return 32768; // Default for 32B+ models
}

// Calculate what fraction of the context window is used.
// Returns a value 0.0 to 1.0.
function calculateContextFill(messages, model) {
  const tokens = estimateMessagesTokens(messages);
  const window = getContextWindow(model);
  return tokens / window;
}

// Context management thresholds (from SenecaChat, based on empirical research
// showing model quality degrades before the nominal context limit).
const THRESHOLDS = {
  ROT_ZONE: 0.60,    // 60% — inject [ROT ZONE] banner, be concise
  WARNING: 0.70,     // 70% — inject [WARNING] banner
  COMPACT: 0.80,     // 80% — trigger compaction
  MINIMAL: 0.85,     // 85% — switch to minimal system prompt mode
};

module.exports = { estimateTokens, estimateMessagesTokens, getContextWindow, calculateContextFill, THRESHOLDS };
```

### 11.2 Tool Result Budget

The tool result budget prevents individual tool outputs from consuming too much context. Each tool result is capped at 32K characters. On overflow, the output is truncated using a 60/40 head/tail strategy with an explicit truncation notice. This matches the Claude Code behavior where `applyToolResultBudget()` runs before every API call in the query loop.

### 11.3 Auto-Compact

When the context reaches 80%, the compaction service summarizes the conversation history into a compact representation. It uses the smallest available model (`getModel('compact')`) to generate a summary of old turns, preserving the last 3 turns verbatim. The summary becomes the new "floor" of the message history — it is injected as a system message followed by the preserved turns.

Write `src/services/compact/autoCompact.js`:

```javascript
'use strict';

const { complete } = require('../ollama/client');
const { estimateMessagesTokens, THRESHOLDS } = require('../../utils/context');
const { db } = require('../../db');

// Number of recent turns to keep verbatim after compaction.
// The last 3 turns preserve the model's "formatting momentum" —
// the model copies the style of recent output when generating new output.
// Keeping these turns prevents style drift after compaction.
const KEEP_RECENT_TURNS = 3;

const COMPACTION_SYSTEM_PROMPT = `You are a conversation summarizer for a coding agent session.
Summarize the conversation below into a concise bullet-point summary that preserves:
1. Every file that was read, created, or modified (with their full paths)
2. Every shell command that was run and its key output
3. The current state of any ongoing task
4. Any errors encountered and their resolutions
5. Key decisions made and their rationale

Be extremely dense and factual. No narrative prose. Use <file>, <cmd>, <error>, <decision> tags.
Keep your summary under 1000 words.`;

async function buildCompactMessages({ messages, systemPrompt, toolContext, model }) {
  if (messages.length <= KEEP_RECENT_TURNS * 2) {
    return null; // Not enough history to compact
  }

  // Split: keep the last N turns verbatim, compact the rest
  const keepFrom = Math.max(0, messages.length - KEEP_RECENT_TURNS * 2);
  const toCompact = messages.slice(0, keepFrom);
  const toKeep = messages.slice(keepFrom);

  if (toCompact.length === 0) return null;

  // Serialize the messages to be compacted into a text block
  const historyText = toCompact.map(msg => {
    const role = msg.role === 'assistant' ? 'ASSISTANT' : msg.role === 'tool' ? 'TOOL_RESULT' : 'USER';
    const content = typeof msg.content === 'string' ? msg.content : JSON.stringify(msg.content);
    return `[${role}]\n${content.slice(0, 4000)}`; // Cap individual messages in compaction input
  }).join('\n\n---\n\n');

  try {
    const response = await complete({
      model,
      messages: [
        { role: 'system', content: COMPACTION_SYSTEM_PROMPT },
        { role: 'user', content: historyText },
      ],
    });

    const summary = response?.message?.content || 'Unable to generate summary.';

    // Build the post-compact message array
    const postCompactMessages = [
      {
        role: 'system',
        content: `[COMPACTION SUMMARY — ${toCompact.length} messages compressed]\n\n${summary}`,
      },
      ...toKeep,
    ];

    return {
      messages: postCompactMessages,
      systemMessages: [`Compacted ${toCompact.length} messages → 1 summary`],
    };
  } catch (err) {
    console.error('Compaction failed:', err.message);
    return null; // Return null — caller will proceed without compaction
  }
}

module.exports = { buildCompactMessages };
```

---

## 12. The Skills System

The skills system makes Euclid domain-aware without fine-tuning. It is derived from the `everything-claude-code` zip archive, which contains a collection of `.agents/skills/<name>/SKILL.md` files. Each skill is a structured markdown document with domain knowledge, coding patterns, and best practices for a specific area.

Write `src/services/skill/loader.js` to scan the `skills/` directory and load all available skills:

```javascript
'use strict';

const fs = require('fs');
const path = require('path');

const SKILLS_DIR = path.join(__dirname, '../../../skills');
const skills = new Map(); // name → { name, description, content }

function loadSkills() {
  if (!fs.existsSync(SKILLS_DIR)) {
    fs.mkdirSync(SKILLS_DIR, { recursive: true });
    return;
  }

  for (const dir of fs.readdirSync(SKILLS_DIR)) {
    const skillPath = path.join(SKILLS_DIR, dir, 'SKILL.md');
    if (!fs.existsSync(skillPath)) continue;

    const content = fs.readFileSync(skillPath, 'utf8');
    const descMatch = content.match(/^description:\s*(.+)$/m);
    const description = descMatch?.[1]?.replace(/^["']|["']$/g, '') || dir;

    skills.set(dir, { name: dir, description, content });
  }

  console.log(`Loaded ${skills.size} skill(s): ${[...skills.keys()].join(', ')}`);
}

function getSkill(name) {
  return skills.get(name);
}

function getAllSkills() {
  return [...skills.values()];
}

// Select the most relevant skills for a given user message using keyword matching.
// Returns up to 3 skills sorted by relevance score.
function selectSkills(userMessage, maxSkills = 3) {
  const msgLower = userMessage.toLowerCase();
  const scored = [];

  for (const [name, skill] of skills) {
    let score = 0;
    const tokens = tokenize(skill.description + ' ' + name);
    const msgTokens = tokenize(msgLower);

    for (const token of tokens) {
      if (msgLower.includes(token)) score += 2;
    }
    for (const token of msgTokens) {
      if (skill.content.toLowerCase().includes(token)) score++;
    }

    if (score > 0) scored.push({ skill, score });
  }

  scored.sort((a, b) => b.score - a.score);
  return scored.slice(0, maxSkills).map(s => s.skill);
}

function tokenize(text) {
  return text.toLowerCase().split(/\W+/).filter(t => t.length > 3);
}

module.exports = { loadSkills, getSkill, getAllSkills, selectSkills };
```

Write the skills content files under `skills/`. Each skill file follows this template adapted from the `everything-claude-code` archive:

```markdown
---
name: backend-patterns
description: Backend architecture patterns, API design, database optimization, service layers, middleware for Node.js and Express
---

# Backend Development Patterns

## When to Activate
- Designing REST API endpoints
- Implementing repository, service, or controller layers
- Optimizing database queries
- Adding caching or background jobs
- Structuring error handling and validation

## Repository Pattern
[complete pattern documentation]

## Service Layer
[complete pattern documentation]
```

Create skill files for: `api-design`, `backend-patterns`, `coding-standards`, `e2e-testing`, `frontend-patterns`, `documentation-lookup`, and `eval-harness`.

---

## 13. Multi-Agent and Task System

Euclid's multi-agent system is derived from two sources: the `Task.ts` type system from Claude Code (which defines task types, IDs, and status) and SenecaChat's orchestrate/debate/peer-review patterns (which define the agent specialization model).

### 13.1 Built-in Agent Roster

Euclid ships with six built-in specialist agents. Each agent has a name, a system prompt tuned for its specialty, and an assigned model. The agents are stored in the `agents` table and loaded at startup.

```javascript
// src/agents/builtins.js
// These are the default specialist agents loaded on first boot.
// They can be overridden or extended via POST /api/agents.

const BUILTIN_AGENTS = [
  {
    id: 'coder',
    name: 'Coder',
    type: 'builtin',
    model: '', // Uses main model
    temperature: 0.1,
    system_prompt: `You are an expert software engineer. Your job is to write clean, efficient, well-tested code.
You prefer simple solutions over complex ones. You write code that is easy to understand and maintain.
When given a task, you: (1) understand the requirements completely, (2) plan the implementation,
(3) write the code, (4) verify it works. You always check if the code compiles/runs before reporting done.`,
  },
  {
    id: 'researcher',
    name: 'Researcher',
    type: 'builtin',
    model: '',
    temperature: 0.3,
    system_prompt: `You are a thorough researcher. Your job is to find accurate information and synthesize it clearly.
You search broadly, evaluate source quality, and present findings in a structured format.
You always cite your sources. You distinguish between facts and inferences. You flag uncertainty.`,
  },
  {
    id: 'writer',
    name: 'Writer',
    type: 'builtin',
    model: '',
    temperature: 0.5,
    system_prompt: `You are a skilled technical writer. You produce clear, concise, accurate documentation.
You write for the reader's level of expertise. You use examples. You prefer active voice and short sentences.`,
  },
  {
    id: 'analyst',
    name: 'Analyst',
    type: 'builtin',
    model: '',
    temperature: 0.2,
    system_prompt: `You are a rigorous analyst. You break problems into components, evaluate tradeoffs,
and produce structured analysis. You use data when available. You surface assumptions. You give recommendations.`,
  },
  {
    id: 'critic',
    name: 'Critic',
    type: 'builtin',
    model: '',
    temperature: 0.4,
    system_prompt: `You are a constructive critic. You identify problems, bugs, logical errors, and missed edge cases.
You score issues by severity: critical, major, minor, suggestion. You are specific — no vague feedback.
You always suggest how to fix the issues you find.`,
  },
  {
    id: 'planner',
    name: 'Planner',
    type: 'builtin',
    model: '',
    temperature: 0.2,
    system_prompt: `You are a strategic planner. You decompose complex tasks into ordered, executable steps.
Each step has a clear description, expected output, and success criterion. You identify dependencies.
You flag risks. You produce plans that a code agent can execute without further clarification.`,
  },
];

module.exports = { BUILTIN_AGENTS };
```

### 13.2 Multi-Agent Orchestration

The orchestration endpoint runs a multi-step workflow: first the Planner agent decomposes the task, then each subtask is assigned to the most suitable specialist, then the results are synthesized by the Analyst.

Write `src/routes/agents.js` with the `/api/agents/orchestrate` endpoint that implements this workflow using the `complete()` Ollama client function (not streaming, because orchestration runs in the background and returns results when done).

---

## 14. BM25 RAG and Memory

The RAG system has two parts: document ingestion (chunking and indexing) and retrieval (BM25 search). It is derived directly from SenecaChat's implementation.

Write `src/utils/bm25.js` with the complete BM25 implementation:

```javascript
'use strict';

// BM25 parameters — these are the standard defaults
const BM25_K1 = 1.5; // Term frequency saturation
const BM25_B  = 0.75; // Length normalization

function tokenize(text) {
  return (text || '')
    .toLowerCase()
    .replace(/[^a-z0-9\s]/g, ' ')
    .split(/\s+/)
    .filter(t => t.length > 2 && !STOPWORDS.has(t));
}

function buildFreqMap(tokens) {
  const freq = {};
  for (const t of tokens) freq[t] = (freq[t] || 0) + 1;
  return freq;
}

function bm25Score(queryTokens, docFreq, docLen, avgLen, N, df) {
  let score = 0;
  for (const term of queryTokens) {
    const tf = docFreq[term] || 0;
    if (!tf) continue;
    const idf = Math.log((N - (df[term] || 0) + 0.5) / ((df[term] || 0) + 0.5) + 1);
    const tfNorm = (tf * (BM25_K1 + 1)) / (tf + BM25_K1 * (1 - BM25_B + BM25_B * docLen / avgLen));
    score += idf * tfNorm;
  }
  return score;
}

// Search chunks using BM25. chunks is an array of { text, freq, len, filename, rel_path }
function search(query, chunks, topK = 8) {
  if (!chunks.length) return [];
  const queryTokens = tokenize(query);
  if (!queryTokens.length) return [];

  const N = chunks.length;
  const avgLen = chunks.reduce((s, c) => s + (c.len || 0), 0) / N;

  // Build global document frequency map
  const df = {};
  for (const c of chunks) {
    for (const term of Object.keys(c.freq || {})) {
      df[term] = (df[term] || 0) + 1;
    }
  }

  const scored = chunks
    .map(c => ({
      ...c,
      score: bm25Score(queryTokens, c.freq || {}, c.len || 1, avgLen, N, df),
    }))
    .filter(c => c.score > 0)
    .sort((a, b) => b.score - a.score);

  // Limit to 3 chunks per source file to avoid overwhelming with one document
  const seen = new Map();
  const result = [];
  for (const r of scored) {
    const count = seen.get(r.filename) || 0;
    if (count < 3) {
      result.push(r);
      seen.set(r.filename, count + 1);
    }
    if (result.length >= topK) break;
  }

  // Normalize confidence scores
  const maxScore = result[0]?.score || 1;
  for (const r of result) r.confidence = Math.round((r.score / maxScore) * 100);

  return result;
}

const STOPWORDS = new Set([
  'the', 'and', 'for', 'are', 'but', 'not', 'you', 'all', 'any', 'can',
  'has', 'her', 'was', 'one', 'our', 'out', 'had', 'have', 'this', 'that',
  'with', 'they', 'from', 'been', 'its', 'into', 'than', 'then', 'also',
]);

module.exports = { tokenize, buildFreqMap, bm25Score, search };
```

---

## 15. Permission and Safety Layer

The permission layer is one of the most important parts of Euclid, derived from the Claude Code permission subsystem (`src/utils/permissions/*`). It classifies every tool use, evaluates allow/deny/ask rules, and tracks denials.

Write `src/services/permission/checker.js` with the permission evaluation logic. The permission system must support three modes: `default` (prompt for destructive actions), `plan` (read-only, prompt for writes), and `bypass` (no prompts — auto-allow everything). Never auto-allow in `plan` mode. Always ask in `plan` mode before any write.

---

## 16. The Frontend

Euclid uses the SenecaChat frontend design from `public/index.html` as its base. This is a single-page application with approximately 200,000 characters of HTML, CSS, and JavaScript. You adapt it for Euclid by:

1. Changing all references to "SenecaChat" to "Euclid" in text, class names, and variables
2. Adding Euclid's additional tool displays (file diff viewer, skill activation indicator, permission prompt dialog)
3. Updating the color scheme to a dark geometric aesthetic consistent with the "Euclid" name (Euclidean geometry motif: precise, mathematical, clean)
4. Adding the tool call visualization — each tool use shows its name, input summary, execution time, and result in a collapsible block
5. Adding the session state panel — shows context fill percentage, current model, active skills, and memory namespace counts
6. Keeping all of SenecaChat's existing features: streaming SSE chat, conversation list, memory viewer, document upload, plan tracker, kanban board, multi-agent chat, eval dashboard, and live log viewer

The AI must extract `public/index.html` from the SenecaChat archive and adapt it. Do not rewrite it from scratch — there are 200,000 characters of battle-tested frontend code there.

```bash
# Extract SenecaChat's frontend
unzip -p /path/to/SenecaChat-master.zip SenecaChat-master/public/index.html > public/index.html

# Then make targeted edits for Euclid branding and new features
```

---

## 17. Auth, Rate Limiting, Circuit Breaker

These three infrastructure components are adapted from SenecaChat's `src/auth.js` and `server.js` patterns.

### 17.1 Auth

Write `src/middleware/auth.js` with the same token-based auth as SenecaChat: a persistent token stored in `data/auth_token.txt`, read on startup, passed as `Authorization: Bearer <token>` or `x-euclid-token` header. Use `crypto.timingSafeEqual` for comparison to prevent timing attacks.

### 17.2 Rate Limiting

Three rate limit tiers using `express-rate-limit`:

```javascript
const chatLimiter  = rateLimit({ windowMs: 60000, max: 60 });   // 60/min
const execLimiter  = rateLimit({ windowMs: 60000, max: 120 });  // 120/min
const heavyLimiter = rateLimit({ windowMs: 60000, max: 10 });   // 10/min (orchestration, eval)
```

### 17.3 Circuit Breaker

Write `src/services/ollama/circuitBreaker.js` as a standalone class with three states (`closed`, `open`, `half-open`), configurable threshold and reset timeout, and exponential retry. This is identical to SenecaChat's `CircuitBreaker` class — copy and adapt it.

---

## 18. Observability and Evals

Observability is built into every layer of Euclid. Every tool call is recorded in `tool_calls`. Every permission decision is recorded in `permission_decisions`. Every token count is recorded in `cost_events`. Every error is recorded in `errors_log`. The audit trail in `audit_log` captures every high-level action.

The eval system implements three types of quality measurement derived from SenecaChat:

**LLM-as-judge** (`POST /api/eval/judge`): Calls the judge model to score a response on 5 dimensions: relevance, accuracy, completeness, clarity, and conciseness. Each dimension is scored 0-10. The overall score is the weighted average.

**Regression suites** (`POST /api/eval/suite`): A collection of test cases, each with a prompt and expected keywords in the response. Running a suite evaluates each test case and records pass/fail.

**Drift detection** (`GET /api/eval/drift`): Compares recent response quality scores to a baseline window and flags if quality has dropped significantly.

---

## 19. Production Deployment on Cloud

This section covers the end-to-end production deployment of Euclid on a cloud VM with a remote Ollama instance on a GPU pod.

### 19.1 Cloud Architecture

```
Browser
  │
  ▼
Nginx (reverse proxy, SSL termination, static file serving)
  │
  ▼
Euclid Server (Node.js, port 3001, PM2 managed)
  │
  ▼
Ollama / vLLM (GPU pod, port 11434, accessible via private network or VPN)
  │
  ▼
SQLite WAL (local disk, /data/euclid.db)
```

### 19.2 Nginx Configuration

Create `/etc/nginx/sites-available/euclid`:

```nginx
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;

    ssl_certificate     /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    # SSE requires these headers — disables buffering for streaming
    location /api/chat {
        proxy_pass http://localhost:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 600s;
        proxy_http_version 1.1;
        proxy_set_header Connection '';
        chunked_transfer_encoding on;
    }

    location / {
        proxy_pass http://localhost:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 120s;
    }
}
```

### 19.3 PM2 Process Management

Create `ecosystem.config.js`:

```javascript
module.exports = {
  apps: [{
    name: 'euclid',
    script: 'src/server.js',
    instances: 1,          // Single instance — SQLite is single-writer
    exec_mode: 'fork',
    watch: false,
    max_memory_restart: '512M',
    env_production: {
      NODE_ENV: 'production',
      PORT: 3001,
      OLLAMA_BASE_URL: 'http://your-gpu-pod-ip:11434',
      EUCLID_MAIN_MODEL: 'qwen2.5-coder:32b-instruct-q4_K_M',
      EUCLID_COMPACT_MODEL: 'qwen2.5:3b',
    },
    log_date_format: 'YYYY-MM-DD HH:mm:ss',
    out_file: 'logs/out.log',
    error_file: 'logs/error.log',
    merge_logs: true,
  }],
};
```

Start with: `pm2 start ecosystem.config.js --env production && pm2 save && pm2 startup`

### 19.4 GPU Pod Setup (vLLM for Multi-User)

When you expect more than one concurrent user, replace Ollama with vLLM on the GPU pod. vLLM uses continuous batching — it serves multiple requests with the same loaded weights simultaneously, which Ollama cannot do.

```bash
# On the GPU pod (A100 80GB or equivalent)
pip install vllm

python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-Coder-32B-Instruct-AWQ \
  --quantization awq \
  --tensor-parallel-size 1 \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.90 \
  --port 8000 \
  --host 0.0.0.0

# Then set in Euclid's .env:
# USE_OPENAI_COMPAT=true
# OLLAMA_BASE_URL=http://gpu-pod-ip:8000
```

AWQ quantization is recommended over GPTQ for vLLM because it has better hardware acceleration support on Ampere and later GPUs. It achieves approximately the same quality as Q4_K_M at lower memory overhead.

### 19.5 Environment Variables

Create `.env.production`:

```bash
PORT=3001
NODE_ENV=production

# Inference
OLLAMA_BASE_URL=http://your-gpu-pod-ip:11434
USE_OPENAI_COMPAT=false

# Models
EUCLID_MAIN_MODEL=qwen2.5-coder:32b-instruct-q4_K_M
EUCLID_COMPACT_MODEL=qwen2.5:3b
EUCLID_SUMMARY_MODEL=qwen2.5-coder:7b-instruct-q4_K_M
EUCLID_JUDGE_MODEL=qwen2.5-coder:7b-instruct-q4_K_M
EUCLID_REFLEXION_MODEL=qwen2.5-coder:32b-instruct-q4_K_M
EUCLID_CLASSIFY_MODEL=qwen2.5:3b

# Auth
EUCLID_TOKEN=        # auto-generated on first boot if not set

# Directories
DATA_DIR=./data
UPLOADS_DIR=./uploads
LOGS_DIR=./logs
SKILLS_DIR=./skills

# Limits
MAX_TURNS=50
MAX_OUTPUT_CHARS=32768
MAX_FILE_SIZE_MB=1
RATE_LIMIT_CHAT=60
RATE_LIMIT_EXEC=120
RATE_LIMIT_HEAVY=10
```

---

## 20. Full File Tree

The AI must create every file and directory listed here. This is the authoritative file tree for Euclid. Any file not listed here is an optional addition.

```
euclid/
├── package.json
├── ecosystem.config.js
├── .env.example
├── .gitignore
├── README.md (this file)
│
├── src/
│   ├── server.js                        # Express app entry point
│   │
│   ├── db/
│   │   ├── index.js                     # All DB operations (prepared statements)
│   │   └── schema.js                    # Complete SQL schema (all tables)
│   │
│   ├── middleware/
│   │   ├── auth.js                      # Bearer token authentication
│   │   └── rateLimit.js                 # Rate limit tier definitions
│   │
│   ├── routes/
│   │   ├── chat.js                      # POST /api/chat (SSE streaming)
│   │   ├── exec.js                      # POST /api/exec (shell execution)
│   │   ├── conversations.js             # CRUD /api/conversations
│   │   ├── memory.js                    # CRUD /api/memory
│   │   ├── docs.js                      # POST /api/docs/ingest, search
│   │   ├── agents.js                    # POST /api/agents/orchestrate, debate, peer-review
│   │   ├── plans.js                     # CRUD /api/plans
│   │   ├── evals.js                     # POST /api/eval/judge, suite, drift
│   │   ├── system.js                    # GET /api/health, metrics, audit, logs
│   │   ├── costs.js                     # GET /api/costs (token and cost tracking)
│   │   └── kanban.js                    # GET/POST /api/issues (Kanban board)
│   │
│   ├── services/
│   │   ├── queryEngine.js               # QueryEngine class (session lifecycle)
│   │   ├── queryLoop.js                 # Main agentic while-loop
│   │   ├── toolResultBudget.js          # 32K output cap with head/tail truncation
│   │   │
│   │   ├── ollama/
│   │   │   ├── client.js                # Ollama HTTP client (streaming + non-streaming)
│   │   │   ├── circuitBreaker.js        # 4-failure threshold, 20s recovery
│   │   │   └── stream.js                # NDJSON stream parser
│   │   │
│   │   ├── compact/
│   │   │   ├── autoCompact.js           # LLM-powered history compaction
│   │   │   └── buildPostCompact.js      # Post-compaction message array builder
│   │   │
│   │   ├── rag/
│   │   │   ├── ingest.js                # Document chunking + indexing
│   │   │   └── retrieve.js             # BM25 retrieval at query time
│   │   │
│   │   ├── skill/
│   │   │   ├── loader.js                # Scan skills/ directory, load SKILL.md files
│   │   │   └── selector.js              # Keyword-based skill selection
│   │   │
│   │   ├── task/
│   │   │   └── runner.js                # Background task execution and status tracking
│   │   │
│   │   └── permission/
│   │       └── checker.js               # Tool permission evaluation (allow/ask/deny)
│   │
│   ├── tools/
│   │   ├── index.js                     # Tool registry (loads + exports all tools)
│   │   ├── runner.js                    # Concurrent/sequential tool execution
│   │   │
│   │   ├── bash/
│   │   │   └── bash.js                  # Shell execution with safety scoring
│   │   ├── read/
│   │   │   └── read.js                  # File read with line range support
│   │   ├── write/
│   │   │   └── write.js                 # File write with diff preview
│   │   ├── edit/
│   │   │   └── edit.js                  # Surgical string replacement
│   │   ├── glob/
│   │   │   └── glob.js                  # File pattern matching
│   │   ├── grep/
│   │   │   └── grep.js                  # Content search (ripgrep or fallback)
│   │   ├── ls/
│   │   │   └── ls.js                    # Directory listing
│   │   ├── agent/
│   │   │   └── agentTool.js             # Spawn and communicate with sub-agents
│   │   ├── todo/
│   │   │   └── todo.js                  # Per-session markdown todo checklist
│   │   └── search/
│   │       └── webSearch.js             # Web search (Brave API or SearXNG)
│   │
│   ├── utils/
│   │   ├── systemPrompt.js              # System prompt assembly with context banners
│   │   ├── context.js                   # Token counting and context fill calculation
│   │   ├── bm25.js                      # BM25 scoring algorithm
│   │   ├── modelRouter.js               # Model selection per operation type
│   │   ├── fileStateCache.js            # LRU cache for file read deduplication
│   │   ├── attachments.js               # Memory and skill attachment injection
│   │   └── safety.js                    # Command safety scoring (shared)
│   │
│   └── agents/
│       └── builtins.js                  # Built-in specialist agent definitions
│
├── skills/
│   ├── api-design/
│   │   └── SKILL.md
│   ├── backend-patterns/
│   │   └── SKILL.md
│   ├── coding-standards/
│   │   └── SKILL.md
│   ├── e2e-testing/
│   │   └── SKILL.md
│   ├── frontend-patterns/
│   │   └── SKILL.md
│   ├── documentation-lookup/
│   │   └── SKILL.md
│   └── eval-harness/
│       └── SKILL.md
│
├── public/
│   └── index.html                       # SenecaChat frontend (adapted for Euclid)
│
├── data/                                # Runtime data (gitignored)
│   └── euclid.db                        # SQLite database (auto-created)
│
├── uploads/                             # Uploaded documents (gitignored)
├── logs/                                # PM2 and application logs (gitignored)
│
└── scripts/
    ├── setup-runpod.sh                  # GPU pod Ollama setup
    └── seed-skills.js                   # Load skills from everything-claude-code
```

---

## 21. Step-by-Step Build Commands

The AI must execute every command below in order. Each command is idempotent — it can be run again without breaking anything. Do not skip any step.

```bash
# ── Step 1: Clone and Initialize ────────────────────────────────────────
mkdir euclid && cd euclid
git init
echo "node_modules/\ndata/\nuploads/\nlogs/\n.env" > .gitignore

# ── Step 2: Install Dependencies ────────────────────────────────────────
npm init -y
npm install express better-sqlite3 node-fetch@2 multer \
  helmet express-rate-limit uuid zod morgan compression \
  fast-glob micromatch
npm install --save-dev nodemon

# ── Step 3: Create Directory Structure ──────────────────────────────────
mkdir -p src/{db,middleware,routes,services/{ollama,compact,rag,skill,task,permission},tools/{bash,read,write,edit,glob,grep,ls,agent,todo,search},utils,agents}
mkdir -p skills/{api-design,backend-patterns,coding-standards,e2e-testing,frontend-patterns,documentation-lookup,eval-harness}
mkdir -p public data uploads logs scripts

# ── Step 4: Copy SenecaChat Frontend ────────────────────────────────────
# (Assumes the zip is available; adapt path as needed)
unzip -p SenecaChat-master.zip SenecaChat-master/public/index.html > public/index.html

# ── Step 5: Create All Source Files ─────────────────────────────────────
# (The AI creates each file with its complete implementation as documented above)

# ── Step 6: Copy Skills from everything-claude-code ──────────────────────
# Extract key skills from the zip archive
for skill in api-design backend-patterns coding-standards e2e-testing frontend-patterns; do
  unzip -p everything-claude-code-main.zip \
    "everything-claude-code-main/.agents/skills/$skill/SKILL.md" \
    > "skills/$skill/SKILL.md" 2>/dev/null || echo "Skill $skill not found in zip"
done

# ── Step 7: Verify Database Initialization ──────────────────────────────
node -e "
const Database = require('better-sqlite3');
const { SCHEMA } = require('./src/db/schema');
const db = new Database('./data/euclid.db');
db.exec(SCHEMA);
const tables = db.prepare(\"SELECT name FROM sqlite_master WHERE type='table'\").all();
console.log('Tables created:', tables.map(t => t.name).join(', '));
db.close();
"

# ── Step 8: Test Ollama Connection ──────────────────────────────────────
OLLAMA_BASE_URL="${OLLAMA_BASE_URL:-http://localhost:11434}"
curl -s "$OLLAMA_BASE_URL/api/tags" | node -e "
const d = require('fs').readFileSync('/dev/stdin','utf8');
try {
  const { models } = JSON.parse(d);
  console.log('Ollama connected. Models:', models.map(m=>m.name).join(', ') || '(none pulled)');
} catch { console.error('Ollama not responding at', process.env.OLLAMA_BASE_URL || 'http://localhost:11434'); }
"

# ── Step 9: Start Development Server ────────────────────────────────────
npm run dev
# Server starts at http://localhost:3001
# Auth token printed to console on first boot

# ── Step 10: Verify All Endpoints ───────────────────────────────────────
TOKEN=$(cat data/auth_token.txt)
echo "Token: $TOKEN"

curl -s http://localhost:3001/api/health | node -e "process.stdin.pipe(process.stdout)"
curl -s http://localhost:3001/api/system/stats | node -e "process.stdin.pipe(process.stdout)"
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:3001/api/conversations
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:3001/api/memory
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:3001/api/agents

# ── Step 11: Test a Full Chat Turn ──────────────────────────────────────
curl -N -s \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"Hello Euclid. What can you do?"}],"model":"qwen2.5-coder:7b-instruct-q4_K_M"}' \
  http://localhost:3001/api/chat
# Should stream SSE events to the terminal

# ── Step 12: Production Build ────────────────────────────────────────────
npm install -g pm2
cp .env.example .env.production
# Edit .env.production with your OLLAMA_BASE_URL and model names
NODE_ENV=production pm2 start ecosystem.config.js --env production
pm2 save
pm2 logs euclid --lines 50

# ── Step 13: Optional — Pull Models on Remote Ollama ────────────────────
# Run this on the GPU pod after running scripts/setup-runpod.sh
REMOTE="user@gpu-pod-ip"
ssh $REMOTE "ollama pull qwen2.5-coder:32b-instruct-q4_K_M && echo done"
ssh $REMOTE "ollama pull qwen2.5:3b && echo done"
ssh $REMOTE "ollama list"
```

---

## Appendix A — Reflexion Implementation

Reflexion (Shinn et al., 2022) is a structured self-improvement loop. When a tool fails or the model's response is rated poorly, Reflexion runs a 5-step critique to produce a better answer in the same turn without human intervention.

Write `src/routes/reflect.js`:

```javascript
'use strict';

const express = require('express');
const router = express.Router();
const { complete } = require('../services/ollama/client');
const { getModel } = require('../utils/modelRouter');

const REFLEXION_PROMPT = `You are a self-reflecting AI agent reviewing a failed or poor-quality response.

Follow these 5 steps exactly:

STEP 1 — IDENTIFY FAILURE: What specifically went wrong? Be precise.
STEP 2 — ROOT CAUSE: Why did it go wrong? What assumption was incorrect?
STEP 3 — LESSON: What is the general principle to apply in the future?
STEP 4 — BETTER APPROACH: How should this specific task have been approached?
STEP 5 — IMPROVED RESPONSE: Provide the corrected, improved response.

Format each step with its number and label. After STEP 5, provide ONLY the improved response with no preamble.`;

router.post('/', async (req, res) => {
  const { model, query, response, error, baseUrl } = req.body;

  if (!query || !response) {
    return res.status(400).json({ error: 'query and response are required' });
  }

  try {
    const userContent = [
      `ORIGINAL QUERY:\n${query}`,
      `ORIGINAL RESPONSE:\n${response}`,
      error ? `ERROR/FAILURE REASON:\n${error}` : null,
      `\nNow perform the 5-step Reflexion analysis and provide an improved response.`,
    ].filter(Boolean).join('\n\n---\n\n');

    const result = await complete({
      model: model || getModel('reflexion'),
      messages: [
        { role: 'system', content: REFLEXION_PROMPT },
        { role: 'user', content: userContent },
      ],
    });

    const content = result?.message?.content || '';

    // Extract the improved response after STEP 5
    const step5Match = content.match(/STEP\s*5[^:]*:(.+)$/si);
    const improvedResponse = step5Match
      ? step5Match[1].trim()
      : content.slice(content.lastIndexOf('\n\n') + 2).trim();

    res.json({
      ok: true,
      reflexionAnalysis: content,
      improvedResponse,
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

module.exports = router;
```

---

## Appendix B — System Prompt Architecture

The system prompt is assembled fresh at the start of each turn from multiple components. The assembly order is critical — components earlier in the prompt have more influence on the model's behavior.

```javascript
// src/utils/systemPrompt.js
// The system prompt is assembled in this exact order:
// 1. Core identity and capability declaration
// 2. Working directory and session context
// 3. Active skills (injected if skill selector found matches)
// 4. Memory continuity (injected on first turn of resumed sessions)
// 5. Context health banners (injected at fill thresholds)
// 6. Custom system prompt override (if provided)
// 7. Append section (if provided)

function buildSystemPrompt({ custom, append, cwd, sessionId, contextFill = 0, compacted = false, skills = [], memorySummary = '' }) {
  if (custom) {
    return custom + (append ? '\n\n' + append : '');
  }

  const parts = [];

  // 1. Core identity
  parts.push(`You are Euclid, an expert AI coding agent. You help engineers write, debug, and understand code.

You have access to tools that let you: read and write files, run shell commands, search codebases, manage a todo list, and delegate work to specialist agents. Always use the right tool for the job.

When you are done with a task, say so clearly. Do not make up results — use tools to verify.
Working directory: ${cwd || process.cwd()}`);

  // 2. Active skills
  if (skills.length > 0) {
    parts.push('## Active Skills\n' + skills.map(s => `### ${s.name}\n${s.content.slice(0, 2000)}`).join('\n\n'));
  }

  // 3. Memory continuity
  if (memorySummary) {
    parts.push('## Session Memory\n' + memorySummary);
  }

  // 4. Compaction notice
  if (compacted) {
    parts.push('[NOTE: Conversation history was compacted. The summary above reflects earlier context.]');
  }

  // 5. Context health banners
  const { THRESHOLDS } = require('./context');
  if (contextFill >= THRESHOLDS.MINIMAL) {
    parts.push('[CRITICAL] Context is at 85%+. Be extremely concise. Use tools minimally. Finish the current task and stop.');
  } else if (contextFill >= THRESHOLDS.COMPACT) {
    parts.push('[WARNING] Context is at 80%+. Be concise. Avoid verbose explanations.');
  } else if (contextFill >= THRESHOLDS.ROT_ZONE) {
    parts.push('[ROT ZONE] Context is at 60%+. Prefer direct answers and targeted tool use.');
  }

  // 6. Append
  if (append) parts.push(append);

  return parts.join('\n\n');
}

module.exports = { buildSystemPrompt };
```


---

> **ADDITIONS — Derived from deep analysis of `src.zip` and `everything-claude-code-main.zip`.**
> The sections below extend the original guide with features that exist in the Claude Code source but were not yet specified for Euclid. Read these immediately after the appendices. They are additive — nothing above is changed.

---

## Table of Contents (New Sections)

- [Appendix C — Plan Mode](#appendix-c--plan-mode)
- [Appendix D — Worktree Isolation](#appendix-d--worktree-isolation)
- [Appendix E — Missing Tool Implementations](#appendix-e--missing-tool-implementations)
  - [E.1 AskUserQuestion Tool](#e1-askuserquestion-tool)
  - [E.2 ToolSearch (Deferred Tool Discovery)](#e2-toolsearch-deferred-tool-discovery)
  - [E.3 WebFetch Tool](#e3-webfetch-tool)
  - [E.4 REPL Tool](#e4-repl-tool)
  - [E.5 Sleep Tool](#e5-sleep-tool)
  - [E.6 SendMessage Tool (Inter-Agent)](#e6-sendmessage-tool-inter-agent)
  - [E.7 ScheduleCron Tool](#e7-schedulecron-tool)
- [Appendix F — Task Lifecycle Tools](#appendix-f--task-lifecycle-tools)
- [Appendix G — User-Defined Hooks System](#appendix-g--user-defined-hooks-system)
- [Appendix H — Automated Memory Extraction](#appendix-h--automated-memory-extraction)
- [Appendix I — Token Budget Tracker with Diminishing Returns](#appendix-i--token-budget-tracker-with-diminishing-returns)
- [Appendix J — LSP Integration](#appendix-j--lsp-integration)
- [Appendix K — Jupyter Notebook Support](#appendix-k--jupyter-notebook-support)
- [Appendix L — QueryConfig and Feature Gate Snapshotting](#appendix-l--queryconfig-and-feature-gate-snapshotting)
- [Appendix M — File History and Undo](#appendix-m--file-history-and-undo)
- [Appendix N — MCP Tool Integration](#appendix-n--mcp-tool-integration)
- [Appendix O — Expanded Skills Roster (All 29 Skills)](#appendix-o--expanded-skills-roster-all-29-skills)
- [Appendix P — Scaffold Corrections](#appendix-p--scaffold-corrections)

---

## Appendix C — Plan Mode

**Source**: `src/tools/EnterPlanModeTool/`, `src/tools/ExitPlanModeTool/`, `src/utils/planModeV2.ts`

Plan Mode is a two-phase interaction model where the model is explicitly not allowed to execute tools during the planning phase — it can only read files and ask questions. This prevents the model from taking irreversible actions before the user has approved a plan. Claude Code activates it via `EnterPlanMode` tool call; Euclid replicates this as a session-level permission toggle.

### How Plan Mode Works

When the model calls `enter_plan_mode`, the session's `permissionMode` is set to `'plan'`. In this mode:

- All write tools (`write`, `edit`, `bash`) have `checkPermissions()` return `{ behavior: 'deny', message: 'In plan mode — reads only until plan is approved.' }`
- The model is injected with a system banner: `[PLAN MODE ACTIVE] You are in planning phase. Read files, ask questions, and produce a detailed step-by-step plan. Do NOT execute any writes or shell commands. When your plan is ready, call exit_plan_mode.`
- The model reads the codebase, constructs a numbered plan, then calls `exit_plan_mode`
- After exit, the user reviews the plan in the UI and either approves (mode returns to `'normal'`) or requests revision (mode stays in `'plan'`)

### Plan Mode V2 (Interview Phase)

The `planModeV2.ts` source adds an **interview phase** before planning. In this phase, the model is instructed to ask up to 5 clarifying questions before writing the plan. This resolves ambiguity before any action is taken. Implement this as an optional flag: `?interviewPhase=true` on the `/api/chat` endpoint.

```javascript
// src/tools/plan/enterPlan.js
'use strict';

const { BaseTool } = require('../base');

class EnterPlanModeTool extends BaseTool {
  constructor() {
    super();
    this.name = 'enter_plan_mode';
    this.description = 'Switch to plan mode. In plan mode you may only read files and ask questions. ' +
      'No writes or shell commands are permitted. Produce a detailed numbered plan, ' +
      'then call exit_plan_mode when ready for user review.';
    this.inputSchema = { type: 'object', properties: {}, required: [] };
  }

  isConcurrencySafe() { return false; }
  isReadOnly() { return true; }

  async checkPermissions(input, context) {
    // Plan mode entry is always allowed
    return { behavior: 'allow' };
  }

  async call(input, context) {
    // Set plan mode on the session
    context.setPlanMode(true);

    const interviewBanner = context.interviewPhase
      ? '\n\n[INTERVIEW PHASE] First ask up to 5 clarifying questions before writing the plan.'
      : '';

    return {
      output: `Plan mode activated. You may now read files and explore the codebase. ' +
        'Do NOT write files or run shell commands. ' +
        'When your plan is complete, call exit_plan_mode.${interviewBanner}`,
    };
  }
}

// src/tools/plan/exitPlan.js — the symmetric counterpart
class ExitPlanModeTool extends BaseTool {
  constructor() {
    super();
    this.name = 'exit_plan_mode';
    this.description = 'Exit plan mode and present the plan for user review. ' +
      'Call this only after you have written a complete numbered plan.';
    this.inputSchema = {
      type: 'object',
      properties: {
        plan: {
          type: 'string',
          description: 'The complete numbered plan, in Markdown, ready for user review.',
        },
      },
      required: ['plan'],
    };
  }

  async call(input, context) {
    context.setPlanMode(false);
    // Store the plan for display in the UI
    context.setPendingPlan(input.plan);
    return {
      output: 'Plan submitted for review. Awaiting user approval before execution continues.',
    };
  }
}

module.exports = { EnterPlanModeTool, ExitPlanModeTool };
```

Add `plan_mode` and `pending_plan` fields to the `QueryEngine` session state. Add a `POST /api/sessions/:id/approve-plan` endpoint that transitions the session back to normal mode. Surface the pending plan in the SSE stream as `{ type: 'plan_ready', plan: string }` so the frontend can display it in a styled approval card.

---

## Appendix D — Worktree Isolation

**Source**: `src/tools/EnterWorktreeTool/`, `src/tools/ExitWorktreeTool/`, `src/utils/worktree.ts`

Git worktrees allow multiple working trees attached to the same repository. Euclid uses worktrees to give each parallel agent its own isolated filesystem view — agents can make changes to different branches simultaneously without conflicting with each other or the user's main working copy.

### Why Worktrees Matter

Without worktrees, two parallel agents writing to the same directory will corrupt each other's changes. With worktrees, each agent has its own checkout of a specific branch, and merges happen explicitly.

### Implementation

```javascript
// src/tools/worktree/enterWorktree.js
'use strict';

const { execSync } = require('child_process');
const path = require('path');
const { v4: uuidv4 } = require('uuid');

class EnterWorktreeTool extends BaseTool {
  constructor() {
    super();
    this.name = 'enter_worktree';
    this.description = 'Create and switch to a new git worktree. Use this before making experimental ' +
      'changes that should be isolated from the main branch. The worktree gets its own branch.';
    this.inputSchema = {
      type: 'object',
      properties: {
        name: {
          type: 'string',
          description: 'Optional slug name for the worktree branch (letters, digits, dashes only, max 64 chars). ' +
            'Auto-generated if not provided.',
        },
      },
      required: [],
    };
  }

  isReadOnly() { return false; }
  isConcurrencySafe() { return false; }

  async checkPermissions(input, context) {
    if (context.permissionMode === 'readonly') {
      return { behavior: 'deny', message: 'Worktree creation requires write permissions.' };
    }
    return { behavior: 'allow' };
  }

  async call(input, context) {
    const gitRoot = findGitRoot(context.cwd);
    if (!gitRoot) {
      return { output: 'Error: Not in a git repository. Worktrees require git.' };
    }

    const slug = input.name || `euclid-${uuidv4().slice(0, 8)}`;
    validateSlug(slug); // throws on invalid chars

    const branchName = `euclid/worktree/${slug}`;
    const worktreePath = path.join(gitRoot, '.git', 'euclid-worktrees', slug);

    execSync(`git worktree add -b "${branchName}" "${worktreePath}"`, { cwd: gitRoot });

    // Update the session's working directory to the new worktree
    const prevCwd = context.cwd;
    context.setCwd(worktreePath);

    return {
      output: `Worktree created at ${worktreePath} on branch ${branchName}. ` +
        `CWD is now ${worktreePath}. Previous CWD was ${prevCwd}. ` +
        `Call exit_worktree when done to merge and clean up.`,
      newMessages: [],
    };
  }
}

function findGitRoot(cwd) {
  try {
    return execSync('git rev-parse --show-toplevel', { cwd }).toString().trim();
  } catch { return null; }
}

function validateSlug(s) {
  if (!/^[a-zA-Z0-9._-]{1,64}$/.test(s)) {
    throw new Error(`Invalid worktree slug: "${s}". Use only letters, digits, dots, underscores, dashes. Max 64 chars.`);
  }
}

module.exports = { EnterWorktreeTool };
```

The `ExitWorktreeTool` mirrors the pattern: it resets `context.cwd` to the original path, runs `git worktree remove <path>`, and optionally pushes the branch. Add a `worktrees` table to the database to track active worktrees per session.

---

## Appendix E — Missing Tool Implementations

The following tools exist in `src/tools/` of the Claude Code source but were absent from Section 7.3 of the main guide. All must be implemented as files under `src/tools/`.

### E.1 AskUserQuestion Tool

**Source**: `src/tools/AskUserQuestionTool/`

The `ask_user_question` tool lets the model interrupt the agentic loop to request a human answer before continuing. This is critical for tasks where the model cannot proceed without clarification (e.g., "Which database should I use for this service?"). Unlike a regular assistant message, this tool pauses execution and returns control to the user through the SSE channel.

```javascript
// src/tools/askUser/askUser.js
'use strict';

class AskUserQuestionTool extends BaseTool {
  constructor() {
    super();
    this.name = 'ask_user_question';
    this.description = 'Ask the human user a question that requires their input before you can continue. ' +
      'Use ONLY when you genuinely cannot proceed without the user\'s answer. ' +
      'Do not use this to stall — ask one clear, specific question.';
    this.inputSchema = {
      type: 'object',
      properties: {
        question: {
          type: 'string',
          description: 'The question to ask the user. Be specific. One question only.',
        },
        choices: {
          type: 'array',
          items: { type: 'string' },
          description: 'Optional list of choices to present as a menu.',
        },
      },
      required: ['question'],
    };
  }

  isReadOnly() { return true; }
  isConcurrencySafe() { return false; }

  async call(input, context) {
    // Emit a special SSE event that suspends the loop and prompts the user
    // The query loop handles { type: 'user_question_pending' } by pausing
    // and waiting for a response via POST /api/sessions/:id/answer
    if (context.onProgress) {
      context.onProgress({
        type: 'user_question_pending',
        question: input.question,
        choices: input.choices || [],
        toolUseId: context.currentToolUseId,
      });
    }

    // The loop will not continue until the answer arrives.
    // The answer is injected back as a tool result by the session handler.
    return { output: '__WAITING_FOR_USER__', suspended: true };
  }
}

module.exports = { AskUserQuestionTool };
```

Add `POST /api/sessions/:id/answer` which accepts `{ toolUseId, answer }`, injects the answer into the paused session's tool result queue, and resumes the generator. Store pending questions in `tool_calls` table with `status = 'pending_user'`.

### E.2 ToolSearch (Deferred Tool Discovery)

**Source**: `src/tools/ToolSearchTool/`

The Claude Code source registers a large number of tools but does not expose all of them to the model in the initial prompt — doing so would waste context. Tools marked `shouldDefer: true` are hidden from the initial tool list. The model discovers them by calling `tool_search`.

```javascript
// src/tools/search/toolSearch.js
'use strict';

class ToolSearchTool extends BaseTool {
  constructor() {
    super();
    this.name = 'tool_search';
    this.description = 'Search for available tools by keyword. Use this if you think a tool exists ' +
      'for your task but you don\'t see it in your tool list.';
    this.inputSchema = {
      type: 'object',
      properties: {
        query: {
          type: 'string',
          description: 'Keyword describing the capability you need (e.g., "notebook", "cron", "lsp").',
        },
      },
      required: ['query'],
    };
    // Tool search itself is always visible — it is never deferred.
    this.shouldDefer = false;
  }

  isReadOnly() { return true; }
  isConcurrencySafe() { return true; }

  async call(input, context) {
    const { loadTools } = require('../index');
    const allTools = loadTools(context);
    const q = input.query.toLowerCase();

    const matches = allTools
      .filter(t => t.shouldDefer) // only search deferred tools
      .filter(t =>
        t.name.toLowerCase().includes(q) ||
        t.description.toLowerCase().includes(q)
      )
      .map(t => `${t.name}: ${t.description}`);

    if (!matches.length) {
      return { output: `No deferred tools matched "${input.query}".` };
    }

    // Exposing the tool name is enough — the loop will add it to the active tool list
    // when the model calls it by name.
    return {
      output: `Found ${matches.length} tool(s) matching "${input.query}":\n\n${matches.join('\n')}`,
    };
  }
}

module.exports = { ToolSearchTool };
```

Mark the following tools as `shouldDefer: true` in their constructors: `schedule_cron`, `notebook_edit`, `enter_worktree`, `exit_worktree`, `lsp_query`, `send_message`. The query loop must dynamically add a deferred tool to the active tool list the first time the model calls it by name — search the full registry, not just the initial list.

### E.3 WebFetch Tool

**Source**: `src/tools/WebFetchTool/`

The WebFetch tool fetches a URL and applies a user-provided prompt to the fetched content using the model — this is different from a raw HTTP GET. The model doesn't receive the raw HTML; it receives a model-processed summary of the content, which is far more token-efficient.

```javascript
// src/tools/webFetch/webFetch.js
'use strict';

const { complete } = require('../../services/ollama/client');
const { getModel } = require('../../utils/modelRouter');

const MAX_FETCH_BYTES = 400_000;
const MAX_MARKDOWN_LENGTH = 100_000; // chars sent to model for processing

// Pre-approved hosts that never need permission prompts
const PREAPPROVED_HOSTS = new Set([
  'github.com', 'docs.github.com', 'npmjs.com', 'pkg.go.dev',
  'docs.python.org', 'developer.mozilla.org', 'stackoverflow.com',
  'pypi.org', 'crates.io', 'registry.npmjs.org',
]);

class WebFetchTool extends BaseTool {
  constructor() {
    super();
    this.name = 'web_fetch';
    this.description = 'Fetch a URL and apply a prompt to the content. Use for reading documentation, ' +
      'checking package versions, reading GitHub issues, or accessing any public web resource.';
    this.inputSchema = {
      type: 'object',
      properties: {
        url: { type: 'string', description: 'The full URL to fetch.' },
        prompt: {
          type: 'string',
          description: 'What to extract or summarize from the fetched content.',
        },
      },
      required: ['url', 'prompt'],
    };
    this.shouldDefer = false; // WebFetch is always visible
  }

  isReadOnly() { return true; }
  isConcurrencySafe() { return true; }

  async checkPermissions(input, context) {
    try {
      const { hostname } = new URL(input.url);
      if (PREAPPROVED_HOSTS.has(hostname)) return { behavior: 'allow' };
      if (context.permissionMode === 'bypassPermissions') return { behavior: 'allow' };
      return {
        behavior: 'ask',
        message: `Allow fetching ${hostname}?`,
      };
    } catch {
      return { behavior: 'deny', message: 'Invalid URL.' };
    }
  }

  async call(input, context) {
    const start = Date.now();
    let rawText;
    let status;

    try {
      const res = await fetch(input.url, {
        headers: { 'User-Agent': 'Euclid/1.0 (AI coding agent; +https://github.com/euclid)' },
        signal: AbortSignal.timeout(15_000),
      });
      status = res.status;
      if (!res.ok) return { output: `HTTP ${status}: ${res.statusText}` };

      const buf = await res.arrayBuffer();
      if (buf.byteLength > MAX_FETCH_BYTES) {
        rawText = Buffer.from(buf).toString('utf8', 0, MAX_FETCH_BYTES) + '\n[TRUNCATED]';
      } else {
        rawText = Buffer.from(buf).toString('utf8');
      }
    } catch (err) {
      return { output: `Fetch failed: ${err.message}` };
    }

    // Strip HTML tags for plain text extraction
    const plainText = rawText
      .replace(/<style[^>]*>[\s\S]*?<\/style>/gi, '')
      .replace(/<script[^>]*>[\s\S]*?<\/script>/gi, '')
      .replace(/<[^>]+>/g, ' ')
      .replace(/\s{2,}/g, ' ')
      .slice(0, MAX_MARKDOWN_LENGTH);

    // Apply the user's prompt to the fetched content using a fast model
    const result = await complete({
      model: getModel('compact'), // use fast model for fetch processing
      messages: [
        {
          role: 'user',
          content: `Content from ${input.url}:\n\n${plainText}\n\n---\n\nTask: ${input.prompt}`,
        },
      ],
      options: { temperature: 0.1, num_predict: 2048 },
    });

    const durationMs = Date.now() - start;

    return {
      output: [
        `URL: ${input.url}`,
        `Status: ${status}`,
        `Size: ${rawText.length} chars`,
        `Duration: ${durationMs}ms`,
        `\nResult:\n${result?.message?.content || '(no output)'}`,
      ].join('\n'),
    };
  }
}

module.exports = { WebFetchTool };
```

### E.4 REPL Tool

**Source**: `src/tools/REPLTool/`

The REPL tool runs arbitrary code in an in-process JavaScript evaluation context (not a full shell). It is useful for quick calculations, data transformations, and small scripts that don't need the full bash tool overhead. Unlike bash, the REPL maintains state across calls within the same session — variables set in one REPL call persist to the next.

```javascript
// src/tools/repl/repl.js
'use strict';

const vm = require('vm');

// One REPL context per session, lazily created
const replContexts = new Map(); // sessionId → vm.Context

class REPLTool extends BaseTool {
  constructor() {
    super();
    this.name = 'repl';
    this.description = 'Execute JavaScript code in a persistent REPL. State is maintained across calls ' +
      'within the session. Use for calculations, data processing, and scripting that does not need shell access.';
    this.inputSchema = {
      type: 'object',
      properties: {
        code: { type: 'string', description: 'JavaScript code to execute.' },
        reset: {
          type: 'boolean',
          description: 'If true, clear the REPL context before executing.',
          default: false,
        },
      },
      required: ['code'],
    };
    this.shouldDefer = false;
  }

  isReadOnly() { return false; } // REPL can have side effects
  isConcurrencySafe() { return false; } // shared context

  async checkPermissions(input, context) {
    // REPL cannot touch the filesystem or network by default
    // We give it a sandboxed context without require/fetch
    return { behavior: 'allow' };
  }

  async call(input, context) {
    const sessionId = context.sessionId;

    if (input.reset || !replContexts.has(sessionId)) {
      replContexts.set(sessionId, vm.createContext({
        console: { log: (...a) => logs.push(a.map(String).join(' ')) },
        Math, JSON, Date, Array, Object, String, Number, Boolean, RegExp,
        parseInt, parseFloat, isNaN, isFinite,
        // Note: require, fetch, fs are intentionally omitted for safety
      }));
    }

    const ctx = replContexts.get(sessionId);
    const logs = [];
    ctx.console = { log: (...a) => logs.push(a.map(String).join(' ')) };

    let result;
    try {
      result = vm.runInContext(input.code, ctx, { timeout: 5000 });
    } catch (err) {
      return { output: `Error: ${err.message}` };
    }

    const output = [
      ...logs,
      result !== undefined ? `=> ${JSON.stringify(result, null, 2)}` : '',
    ].filter(Boolean).join('\n');

    return { output: output || '(no output)' };
  }
}

module.exports = { REPLTool };
```

### E.5 Sleep Tool

**Source**: `src/tools/SleepTool/`

The Sleep tool pauses the agentic loop for a specified duration. It is used by the model when it needs to wait for an external operation to complete — a deployment to propagate, a build server to finish, a rate limit to reset.

```javascript
// src/tools/sleep/sleep.js
'use strict';

const MAX_SLEEP_MS = 60_000; // 60 seconds max to prevent runaway loops

class SleepTool extends BaseTool {
  constructor() {
    super();
    this.name = 'sleep';
    this.description = 'Pause execution for a specified duration. Use when waiting for ' +
      'external operations (deploys, builds, rate limits). Max 60 seconds.';
    this.inputSchema = {
      type: 'object',
      properties: {
        seconds: {
          type: 'number',
          description: 'Duration to sleep in seconds (max 60).',
        },
        reason: {
          type: 'string',
          description: 'Why you are sleeping. Shown to the user.',
        },
      },
      required: ['seconds', 'reason'],
    };
    this.shouldDefer = true; // Discovered via tool_search
  }

  isReadOnly() { return true; }
  isConcurrencySafe() { return true; }

  async call(input, context) {
    const ms = Math.min(Math.max(input.seconds * 1000, 0), MAX_SLEEP_MS);

    if (context.onProgress) {
      context.onProgress({
        type: 'system',
        message: `Sleeping ${input.seconds}s: ${input.reason}`,
      });
    }

    await new Promise(resolve => setTimeout(resolve, ms));
    return { output: `Slept ${ms / 1000}s. Reason: ${input.reason}` };
  }
}

module.exports = { SleepTool };
```

### E.6 SendMessage Tool (Inter-Agent)

**Source**: `src/tools/SendMessageTool/`

The `send_message` tool allows a sub-agent to send a structured message to another named agent or back to the orchestrator. This is the communication primitive for multi-agent coordination — rather than one agent calling another inline (which blocks), `send_message` writes to the `agent_messages` table and optionally wakes the target agent.

```javascript
// src/tools/sendMessage/sendMessage.js
'use strict';

const { db } = require('../../db');
const { v4: uuidv4 } = require('uuid');

class SendMessageTool extends BaseTool {
  constructor() {
    super();
    this.name = 'send_message';
    this.description = 'Send a structured message to another agent or the orchestrator. ' +
      'Use for asynchronous multi-agent coordination.';
    this.inputSchema = {
      type: 'object',
      properties: {
        to: { type: 'string', description: 'Target agent name or "orchestrator".' },
        subject: { type: 'string', description: 'Short subject line.' },
        body: { type: 'string', description: 'Message body.' },
        priority: {
          type: 'string',
          enum: ['low', 'normal', 'high', 'urgent'],
          default: 'normal',
        },
      },
      required: ['to', 'subject', 'body'],
    };
    this.shouldDefer = true;
  }

  isReadOnly() { return false; }
  isConcurrencySafe() { return true; }

  async call(input, context) {
    const id = uuidv4();
    db.prepare(`
      INSERT INTO agent_messages (id, from_agent, to_agent, subject, body, priority, session_id, created_at)
      VALUES (?, ?, ?, ?, ?, ?, ?, ?)
    `).run(id, context.agentId || 'main', input.to, input.subject, input.body,
           input.priority || 'normal', context.sessionId, Date.now());

    return {
      output: `Message sent to "${input.to}" (id: ${id}). Subject: "${input.subject}".`,
    };
  }
}

module.exports = { SendMessageTool };
```

Add an `agent_messages` table to the schema:

```sql
CREATE TABLE IF NOT EXISTS agent_messages (
  id TEXT PRIMARY KEY,
  from_agent TEXT NOT NULL,
  to_agent TEXT NOT NULL,
  subject TEXT NOT NULL,
  body TEXT NOT NULL,
  priority TEXT DEFAULT 'normal',
  status TEXT DEFAULT 'unread' CHECK(status IN ('unread','read','archived')),
  session_id TEXT DEFAULT '',
  created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);
CREATE INDEX IF NOT EXISTS idx_agent_messages_to ON agent_messages(to_agent, status, created_at DESC);
```

### E.7 ScheduleCron Tool

**Source**: `src/tools/ScheduleCronTool/`

The `schedule_cron` tool registers a recurring task with a cron expression. The task runs on the server as a background shell command on the specified schedule. Results are written to the `tasks` table for the agent to poll.

```javascript
// src/tools/cron/scheduleCron.js
'use strict';

const { db } = require('../../db');
const { v4: uuidv4 } = require('uuid');

// Validate a basic cron expression (5 or 6 fields)
function isValidCron(expr) {
  const parts = expr.trim().split(/\s+/);
  return parts.length === 5 || parts.length === 6;
}

class ScheduleCronTool extends BaseTool {
  constructor() {
    super();
    this.name = 'schedule_cron';
    this.description = 'Schedule a recurring shell command on a cron schedule. ' +
      'Results are stored in the tasks table.';
    this.inputSchema = {
      type: 'object',
      properties: {
        name: { type: 'string', description: 'Unique name for this cron job.' },
        schedule: {
          type: 'string',
          description: 'Cron expression, e.g. "*/5 * * * *" for every 5 minutes.',
        },
        command: { type: 'string', description: 'Shell command to run.' },
        maxRuns: {
          type: 'integer',
          description: 'Max number of times to run (0 = unlimited).',
          default: 0,
        },
      },
      required: ['name', 'schedule', 'command'],
    };
    this.shouldDefer = true;
  }

  isReadOnly() { return false; }
  isConcurrencySafe() { return false; }

  async checkPermissions(input, context) {
    // Scheduling recurring commands is always destructive-class
    if (context.permissionMode === 'readonly') {
      return { behavior: 'deny', message: 'Cannot schedule cron jobs in readonly mode.' };
    }
    return { behavior: 'ask', message: `Schedule cron "${input.name}" (${input.schedule}): ${input.command}?` };
  }

  async call(input, context) {
    if (!isValidCron(input.schedule)) {
      return { output: `Invalid cron expression: "${input.schedule}". Use 5 or 6 space-separated fields.` };
    }

    const id = `c${uuidv4().replace(/-/g, '').slice(0, 8)}`;

    db.prepare(`
      INSERT INTO scheduled_jobs (id, name, schedule, command, max_runs, run_count, session_id, enabled, created_at)
      VALUES (?, ?, ?, ?, ?, 0, ?, 1, ?)
    `).run(id, input.name, input.schedule, input.command, input.maxRuns || 0, context.sessionId, Date.now());

    return {
      output: `Cron job "${input.name}" registered (id: ${id}). ` +
        `Schedule: ${input.schedule}. Command: ${input.command}. ` +
        `Max runs: ${input.maxRuns || 'unlimited'}.`,
    };
  }
}

module.exports = { ScheduleCronTool };
```

Add `scheduled_jobs` to the schema and wire a `node-cron` or `node-schedule` based runner in `src/services/cron/runner.js` that loads all enabled jobs from the DB at startup. Add this table:

```sql
CREATE TABLE IF NOT EXISTS scheduled_jobs (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,
  schedule TEXT NOT NULL,
  command TEXT NOT NULL,
  max_runs INTEGER DEFAULT 0,
  run_count INTEGER DEFAULT 0,
  last_run_at INTEGER,
  last_result TEXT DEFAULT '',
  session_id TEXT DEFAULT '',
  enabled INTEGER DEFAULT 1,
  created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);
```

Install the scheduler: `npm install node-cron`.

---

## Appendix F — Task Lifecycle Tools

**Source**: `src/tools/TaskCreateTool/`, `TaskGetTool/`, `TaskListTool/`, `TaskOutputTool/`, `TaskStopTool/`, `TaskUpdateTool/`

Section 13 of the main guide describes the `Task.ts` type system but does not implement the six tools that manipulate task lifecycle from within the agentic loop. These tools are what allow the model to spawn background tasks, monitor their status, and read their output without blocking the main loop.

### Tool Summary

| Tool | Name | Purpose |
|------|------|---------|
| `TaskCreateTool` | `create_task` | Spawn a new background task (bash or sub-agent) |
| `TaskGetTool` | `get_task` | Get the current status of a task by ID |
| `TaskListTool` | `list_tasks` | List all active tasks for the current session |
| `TaskOutputTool` | `get_task_output` | Read the current output of a running task |
| `TaskStopTool` | `stop_task` | Send a kill signal to a running task |
| `TaskUpdateTool` | `update_task` | Update a task's description or metadata |

### Full Implementation

```javascript
// src/tools/task/taskTools.js
'use strict';

const { db } = require('../../db');
const { v4: uuidv4 } = require('uuid');
const { spawn } = require('child_process');
const fs = require('fs');
const path = require('path');

// Active task processes — keyed by task ID
const activeProcesses = new Map();

// Task ID format: type prefix + 8 random alphanumeric chars
// Matches the Claude Code generateTaskId() from Task.ts
function generateTaskId(type) {
  const ALPHABET = 'abcdefghijklmnopqrstuvwxyz0123456789';
  const random = Array.from({ length: 8 }, () =>
    ALPHABET[Math.floor(Math.random() * ALPHABET.length)]
  ).join('');
  const prefix = { bash: 'b', agent: 'a', workflow: 'w', monitor: 'm' }[type] || 'x';
  return `${prefix}${random}`;
}

class CreateTaskTool extends BaseTool {
  constructor() {
    super();
    this.name = 'create_task';
    this.description = 'Spawn a background task. For type "bash", runs a shell command asynchronously. ' +
      'For type "agent", runs a sub-agent with a given prompt. Returns a task ID to poll with get_task.';
    this.inputSchema = {
      type: 'object',
      properties: {
        type: { type: 'string', enum: ['bash', 'agent'], description: 'Task type.' },
        command: { type: 'string', description: 'Shell command (for type "bash").' },
        prompt: { type: 'string', description: 'Task description for sub-agent (for type "agent").' },
        description: { type: 'string', description: 'Human-readable task description.' },
      },
      required: ['type', 'description'],
    };
  }

  isConcurrencySafe() { return true; }
  isReadOnly() { return false; }

  async call(input, context) {
    const taskId = generateTaskId(input.type);
    const outputPath = path.join(context.logsDir || '/tmp', `task-${taskId}.log`);

    db.prepare(`
      INSERT INTO tasks (id, type, description, status, output_path, session_id, created_at)
      VALUES (?, ?, ?, 'pending', ?, ?, ?)
    `).run(taskId, input.type, input.description, outputPath, context.sessionId, Date.now());

    if (input.type === 'bash') {
      if (!input.command) return { output: 'Error: bash tasks require a command.' };

      const proc = spawn('bash', ['-c', input.command], {
        cwd: context.cwd,
        stdio: ['ignore', 'pipe', 'pipe'],
      });

      const logStream = fs.createWriteStream(outputPath, { flags: 'a' });
      proc.stdout.pipe(logStream);
      proc.stderr.pipe(logStream);

      activeProcesses.set(taskId, proc);
      db.prepare(`UPDATE tasks SET status = 'running', started_at = ? WHERE id = ?`).run(Date.now(), taskId);

      proc.on('exit', (code) => {
        const status = code === 0 ? 'completed' : 'failed';
        db.prepare(`UPDATE tasks SET status = ?, exit_code = ?, completed_at = ? WHERE id = ?`)
          .run(status, code, Date.now(), taskId);
        activeProcesses.delete(taskId);
        logStream.end();
      });

      return {
        output: `Task ${taskId} started (bash). PID: ${proc.pid}. ` +
          `Output writing to ${outputPath}. Poll with get_task("${taskId}").`,
      };
    }

    if (input.type === 'agent') {
      // Sub-agent tasks are queued and picked up by the task runner service
      db.prepare(`UPDATE tasks SET status = 'queued', prompt = ? WHERE id = ?`)
        .run(input.prompt || input.description, taskId);

      return {
        output: `Task ${taskId} queued (agent). ` +
          `Prompt: "${(input.prompt || input.description).slice(0, 100)}...". ` +
          `Poll with get_task("${taskId}").`,
      };
    }

    return { output: `Unknown task type: ${input.type}` };
  }
}

class GetTaskTool extends BaseTool {
  constructor() {
    super();
    this.name = 'get_task';
    this.description = 'Get the current status of a task by ID.';
    this.inputSchema = {
      type: 'object',
      properties: { task_id: { type: 'string' } },
      required: ['task_id'],
    };
  }

  isReadOnly() { return true; }
  isConcurrencySafe() { return true; }

  async call(input) {
    const task = db.prepare(`SELECT * FROM tasks WHERE id = ?`).get(input.task_id);
    if (!task) return { output: `Task "${input.task_id}" not found.` };

    return {
      output: [
        `ID: ${task.id}`,
        `Type: ${task.type}`,
        `Status: ${task.status}`,
        `Description: ${task.description}`,
        task.exit_code != null ? `Exit code: ${task.exit_code}` : '',
        task.started_at ? `Started: ${new Date(task.started_at).toISOString()}` : '',
        task.completed_at ? `Completed: ${new Date(task.completed_at).toISOString()}` : '',
        `Output file: ${task.output_path}`,
      ].filter(Boolean).join('\n'),
    };
  }
}

class ListTasksTool extends BaseTool {
  constructor() {
    super();
    this.name = 'list_tasks';
    this.description = 'List all tasks for the current session.';
    this.inputSchema = {
      type: 'object',
      properties: {
        status: {
          type: 'string',
          enum: ['all', 'pending', 'running', 'completed', 'failed', 'killed'],
          default: 'all',
        },
      },
      required: [],
    };
  }

  isReadOnly() { return true; }
  isConcurrencySafe() { return true; }

  async call(input, context) {
    const filter = input.status && input.status !== 'all' ? `AND status = '${input.status}'` : '';
    const tasks = db.prepare(
      `SELECT id, type, status, description, created_at FROM tasks WHERE session_id = ? ${filter} ORDER BY created_at DESC LIMIT 50`
    ).all(context.sessionId);

    if (!tasks.length) return { output: 'No tasks found.' };

    const lines = tasks.map(t =>
      `[${t.status.toUpperCase()}] ${t.id} (${t.type}): ${t.description.slice(0, 60)}`
    );
    return { output: lines.join('\n') };
  }
}

class GetTaskOutputTool extends BaseTool {
  constructor() {
    super();
    this.name = 'get_task_output';
    this.description = 'Read the current stdout/stderr output of a task. ' +
      'For running tasks, returns the latest output. For completed tasks, returns the full output.';
    this.inputSchema = {
      type: 'object',
      properties: {
        task_id: { type: 'string' },
        tail: {
          type: 'integer',
          description: 'Number of lines from the end to return (default: 100).',
          default: 100,
        },
      },
      required: ['task_id'],
    };
  }

  isReadOnly() { return true; }
  isConcurrencySafe() { return true; }

  async call(input) {
    const task = db.prepare(`SELECT output_path, status FROM tasks WHERE id = ?`).get(input.task_id);
    if (!task) return { output: `Task "${input.task_id}" not found.` };

    try {
      const content = fs.readFileSync(task.output_path, 'utf8');
      const lines = content.split('\n');
      const tail = input.tail || 100;
      const result = lines.slice(-tail).join('\n');
      return {
        output: `[Task ${input.task_id} | Status: ${task.status} | Lines ${Math.max(0, lines.length - tail)}-${lines.length}]\n\n${result}`,
      };
    } catch {
      return { output: `No output yet for task "${input.task_id}".` };
    }
  }
}

class StopTaskTool extends BaseTool {
  constructor() {
    super();
    this.name = 'stop_task';
    this.description = 'Send a kill signal to a running task.';
    this.inputSchema = {
      type: 'object',
      properties: {
        task_id: { type: 'string' },
        signal: {
          type: 'string',
          enum: ['SIGTERM', 'SIGKILL'],
          default: 'SIGTERM',
          description: 'Signal to send. Use SIGTERM first; SIGKILL only if it does not stop.',
        },
      },
      required: ['task_id'],
    };
  }

  isReadOnly() { return false; }
  isConcurrencySafe() { return false; }
  isDestructive() { return true; }

  async call(input) {
    const proc = activeProcesses.get(input.task_id);
    if (!proc) {
      return { output: `Task "${input.task_id}" is not a running bash task or was already stopped.` };
    }

    proc.kill(input.signal || 'SIGTERM');
    db.prepare(`UPDATE tasks SET status = 'killed', completed_at = ? WHERE id = ?`)
      .run(Date.now(), input.task_id);

    return { output: `Sent ${input.signal || 'SIGTERM'} to task ${input.task_id}.` };
  }
}

module.exports = { CreateTaskTool, GetTaskTool, ListTasksTool, GetTaskOutputTool, StopTaskTool };
```

Update the `tasks` table schema to add the missing columns:

```sql
ALTER TABLE tasks ADD COLUMN prompt TEXT DEFAULT '';
ALTER TABLE tasks ADD COLUMN exit_code INTEGER;
ALTER TABLE tasks ADD COLUMN started_at INTEGER;
ALTER TABLE tasks ADD COLUMN completed_at INTEGER;
```

Or add them to the initial schema definition in `src/db/schema.js`.

---

## Appendix G — User-Defined Hooks System

**Source**: `src/utils/hooks.ts`, `src/types/hooks.ts`, `src/schemas/hooks.ts`

The Claude Code hooks system allows users to register shell commands that fire at specific points in the agent lifecycle. Euclid must implement the same four hook types:

| Hook | Fires When |
|------|-----------|
| `PreToolUse` | Before any tool executes — can block the tool call |
| `PostToolUse` | After any tool completes — can modify the result |
| `Stop` | After the model produces a final response — for memory extraction, notifications |
| `Notification` | On any significant event (errors, completions, user questions) |

### Hook Configuration

Hooks are configured in `.euclid/hooks.json` in the project root (or `~/.euclid/hooks.json` for global hooks):

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": { "tool_name": "bash" },
        "command": "/path/to/my-security-checker.sh",
        "timeout_ms": 5000
      }
    ],
    "PostToolUse": [
      {
        "matcher": { "tool_name": "file_write" },
        "command": "prettier --write ${EUCLID_TOOL_INPUT_path}",
        "timeout_ms": 10000
      }
    ],
    "Stop": [
      {
        "matcher": null,
        "command": "~/.euclid/hooks/extract-memories.sh",
        "timeout_ms": 30000
      }
    ]
  }
}
```

### Hook Environment Variables

Every hook receives these variables in its environment:

```
EUCLID_SESSION_ID       — current session ID
EUCLID_TOOL_NAME        — tool being called (PreToolUse/PostToolUse only)
EUCLID_TOOL_INPUT       — JSON-encoded tool input
EUCLID_TOOL_RESULT      — JSON-encoded tool result (PostToolUse only)
EUCLID_HOOK_TYPE        — hook type (PreToolUse, PostToolUse, Stop, Notification)
EUCLID_CWD              — current working directory
EUCLID_TURN_COUNT       — current turn number in the agentic loop
```

### Hook Runner Implementation

```javascript
// src/services/hooks/hookRunner.js
'use strict';

const { execFile } = require('child_process');
const fs = require('fs');
const path = require('path');

let _hooksConfig = null;

function loadHooksConfig(cwd) {
  // Look for hooks config in project dir, then home dir
  const locations = [
    path.join(cwd, '.euclid', 'hooks.json'),
    path.join(process.env.HOME || '', '.euclid', 'hooks.json'),
  ];

  for (const loc of locations) {
    if (fs.existsSync(loc)) {
      try {
        return JSON.parse(fs.readFileSync(loc, 'utf8'));
      } catch { /* continue */ }
    }
  }
  return { hooks: {} };
}

function matchesHook(hookConfig, toolName) {
  if (!hookConfig.matcher) return true; // null matcher = match all
  if (hookConfig.matcher.tool_name) return hookConfig.matcher.tool_name === toolName;
  return true;
}

/**
 * Run PreToolUse hooks. Returns { block: boolean, reason: string } if a hook
 * wants to block the tool call; returns { block: false } to allow it.
 */
async function runPreToolUseHooks(toolName, toolInput, context) {
  if (!_hooksConfig) _hooksConfig = loadHooksConfig(context.cwd);
  const hooks = (_hooksConfig.hooks || {}).PreToolUse || [];

  for (const hook of hooks) {
    if (!matchesHook(hook, toolName)) continue;

    const output = await runHookCommand(hook.command, {
      EUCLID_HOOK_TYPE: 'PreToolUse',
      EUCLID_TOOL_NAME: toolName,
      EUCLID_TOOL_INPUT: JSON.stringify(toolInput),
      EUCLID_SESSION_ID: context.sessionId,
      EUCLID_CWD: context.cwd,
    }, hook.timeout_ms || 5000);

    // If the hook exits with non-zero or outputs JSON { block: true, reason: "..." },
    // the tool call is blocked.
    if (output.exitCode !== 0) {
      return { block: true, reason: output.stderr || `PreToolUse hook exited ${output.exitCode}` };
    }

    try {
      const parsed = JSON.parse(output.stdout);
      if (parsed.block) return { block: true, reason: parsed.reason || 'Blocked by hook.' };
    } catch { /* non-JSON output is fine — just means "allow" */ }
  }

  return { block: false };
}

/**
 * Run PostToolUse hooks. Hooks can modify the tool result by outputting JSON { result: "..." }.
 */
async function runPostToolUseHooks(toolName, toolInput, toolResult, context) {
  if (!_hooksConfig) _hooksConfig = loadHooksConfig(context.cwd);
  const hooks = (_hooksConfig.hooks || {}).PostToolUse || [];

  let currentResult = toolResult;

  for (const hook of hooks) {
    if (!matchesHook(hook, toolName)) continue;

    const output = await runHookCommand(hook.command, {
      EUCLID_HOOK_TYPE: 'PostToolUse',
      EUCLID_TOOL_NAME: toolName,
      EUCLID_TOOL_INPUT: JSON.stringify(toolInput),
      EUCLID_TOOL_RESULT: JSON.stringify(currentResult),
      EUCLID_SESSION_ID: context.sessionId,
      EUCLID_CWD: context.cwd,
    }, hook.timeout_ms || 10000);

    try {
      const parsed = JSON.parse(output.stdout);
      if (parsed.result !== undefined) currentResult = parsed.result;
    } catch { /* ignore non-JSON output */ }
  }

  return currentResult;
}

/**
 * Run Stop hooks. These fire after the model produces a final response.
 * Stop hooks can inject messages back into the conversation (for memory extraction etc).
 */
async function runStopHooks(context) {
  if (!_hooksConfig) _hooksConfig = loadHooksConfig(context.cwd);
  const hooks = (_hooksConfig.hooks || {}).Stop || [];
  const injectedMessages = [];

  for (const hook of hooks) {
    const output = await runHookCommand(hook.command, {
      EUCLID_HOOK_TYPE: 'Stop',
      EUCLID_SESSION_ID: context.sessionId,
      EUCLID_CWD: context.cwd,
      EUCLID_TURN_COUNT: String(context.turnCount || 0),
    }, hook.timeout_ms || 30000);

    try {
      const parsed = JSON.parse(output.stdout);
      if (parsed.inject_message) injectedMessages.push(parsed.inject_message);
    } catch { /* ignore */ }
  }

  return injectedMessages;
}

function runHookCommand(command, env, timeoutMs) {
  return new Promise((resolve) => {
    const child = execFile('bash', ['-c', command], {
      env: { ...process.env, ...env },
      timeout: timeoutMs,
    }, (error, stdout, stderr) => {
      resolve({
        exitCode: error?.code || 0,
        stdout: stdout || '',
        stderr: stderr || '',
      });
    });
  });
}

module.exports = { runPreToolUseHooks, runPostToolUseHooks, runStopHooks, loadHooksConfig };
```

Wire `runPreToolUseHooks` into `executeTool()` in `src/tools/runner.js` — call it before `validateInput`. Wire `runPostToolUseHooks` after `tool.call()`. Wire `runStopHooks` in `queryLoop.js` after the loop exits cleanly (no tool calls).

---

## Appendix H — Automated Memory Extraction

**Source**: `src/services/extractMemories/extractMemories.ts`, `src/utils/forkedAgent.ts`

The Claude Code source runs an automated memory extraction at the end of every complete query loop turn (when the model produces a final response). This extraction uses a **forked agent** — a separate model call that shares the parent's prompt cache, making it nearly free in terms of TTFT.

### What Gets Extracted

The extractor reads the conversation transcript and identifies:
- Key facts established in the session (library versions, architecture decisions, user preferences)
- Important file paths that were created, modified, or referenced
- Errors encountered and how they were resolved
- Tasks completed and their outcomes

Extracted memories are written to `~/.euclid/projects/<project-hash>/memory/*.md`. On the first turn of the next session, these memory files are scanned and injected into the system prompt via the `memorySummary` field in `buildSystemPrompt`.

### Forked Agent Pattern

A forked agent shares the parent agent's exact system prompt, tool list, and message prefix — this guarantees the prompt cache is hit (Anthropic/Ollama caches by matching the full prefix). Euclid's memory extractor does not need a perfect cache hit since it's calling Ollama, but the pattern is still worth preserving for code clarity.

```javascript
// src/services/memory/extractMemories.js
'use strict';

const { complete } = require('../ollama/client');
const { getModel } = require('../../utils/modelRouter');
const { db } = require('../../db');
const fs = require('fs');
const path = require('path');
const crypto = require('crypto');
const { v4: uuidv4 } = require('uuid');

const EXTRACT_MEMORIES_PROMPT = `You are a memory extraction assistant. Your job is to read a conversation transcript
and extract durable facts that should be remembered in future sessions.

Extract only facts that are:
1. Specific and verifiable (not vague impressions)
2. Likely to be relevant in a future session on the same project
3. Not already obvious from the code itself

Output a JSON object with this exact shape:
{
  "facts": ["<fact 1>", "<fact 2>"],
  "file_paths": ["<path 1>", "<path 2>"],
  "errors_resolved": ["<error and resolution>"],
  "decisions": ["<architectural/design decision made>"]
}

Output ONLY the JSON. No preamble, no explanation.`;

/**
 * Run memory extraction after a completed query loop turn.
 * This is called by the Stop hook handler in stopHooks.js.
 *
 * @param {Array} messages - Full conversation messages for this turn
 * @param {Object} context - Session context
 */
async function extractMemories(messages, context) {
  // Only extract every N turns to avoid overhead (configurable)
  const extractEvery = parseInt(process.env.MEMORY_EXTRACT_EVERY || '3', 10);
  const turnCount = context.turnCount || 0;
  if (turnCount % extractEvery !== 0) return;

  // Build a compact transcript
  const transcript = messages
    .filter(m => m.role === 'user' || m.role === 'assistant')
    .slice(-20) // last 20 messages only
    .map(m => `[${m.role.toUpperCase()}]: ${typeof m.content === 'string' ? m.content : JSON.stringify(m.content)}`)
    .join('\n\n');

  let extracted;
  try {
    const result = await complete({
      model: getModel('compact'), // Use the fast compact model
      messages: [
        { role: 'system', content: EXTRACT_MEMORIES_PROMPT },
        { role: 'user', content: `TRANSCRIPT:\n${transcript}` },
      ],
      options: { temperature: 0.1, num_predict: 1024 },
    });

    const content = result?.message?.content || '';
    // Strip any markdown fences
    const json = content.replace(/```json\s*/g, '').replace(/```/g, '').trim();
    extracted = JSON.parse(json);
  } catch (err) {
    // Memory extraction failure is non-fatal — log and continue
    console.error('[memory] extraction failed:', err.message);
    return;
  }

  if (!extracted || (!extracted.facts?.length && !extracted.decisions?.length)) return;

  // Persist to DB
  const timestamp = Date.now();
  const memoryId = uuidv4();

  db.prepare(`
    INSERT INTO memory (id, namespace, content, session_id, created_at)
    VALUES (?, 'auto', ?, ?, ?)
  `).run(memoryId, JSON.stringify(extracted), context.sessionId, timestamp);

  // Also write to filesystem for persistence across DB resets
  const projectHash = crypto.createHash('sha256')
    .update(context.cwd || process.cwd()).digest('hex').slice(0, 16);
  const memDir = path.join(
    process.env.HOME || '', '.euclid', 'projects', projectHash, 'memory'
  );
  fs.mkdirSync(memDir, { recursive: true });

  const memFile = path.join(memDir, `${timestamp}-${memoryId.slice(0, 8)}.json`);
  fs.writeFileSync(memFile, JSON.stringify(extracted, null, 2), 'utf8');
}

/**
 * Load all memory files for a project and format them as a summary string
 * to inject into the system prompt at session start.
 */
function loadMemorySummary(cwd) {
  const projectHash = crypto.createHash('sha256')
    .update(cwd || process.cwd()).digest('hex').slice(0, 16);
  const memDir = path.join(
    process.env.HOME || '', '.euclid', 'projects', projectHash, 'memory'
  );

  if (!fs.existsSync(memDir)) return '';

  const files = fs.readdirSync(memDir)
    .filter(f => f.endsWith('.json'))
    .sort()
    .slice(-10); // last 10 memory files

  const memories = [];
  for (const file of files) {
    try {
      const data = JSON.parse(fs.readFileSync(path.join(memDir, file), 'utf8'));
      if (data.facts?.length) memories.push(...data.facts.map(f => `• ${f}`));
      if (data.decisions?.length) memories.push(...data.decisions.map(d => `• [DECISION] ${d}`));
    } catch { /* skip corrupt files */ }
  }

  if (!memories.length) return '';
  return `## Recalled from Previous Sessions\n${memories.join('\n')}`;
}

module.exports = { extractMemories, loadMemorySummary };
```

Wire `extractMemories()` into the stop hook handler: after `runStopHooks()` in `queryLoop.js`, call `extractMemories(state.messages, toolContext)`. Wire `loadMemorySummary()` into `QueryEngine.submitMessage()` on the first turn of a resumed session.

---

## Appendix I — Token Budget Tracker with Diminishing Returns

**Source**: `src/query/tokenBudget.ts`

The Claude Code source implements a `BudgetTracker` that monitors how many tokens are being used across continuation turns. It detects **diminishing returns** — when the model is producing fewer and fewer tokens per continuation attempt — and stops continuing even if the token budget has not been exhausted. This prevents infinite continuation loops on stuck models.

```javascript
// src/utils/tokenBudget.js
'use strict';

// Matches the Claude Code constants from tokenBudget.ts
const COMPLETION_THRESHOLD = 0.90; // stop if turn tokens >= 90% of budget
const DIMINISHING_THRESHOLD = 500; // tokens/continuation — below this = diminishing returns
const MAX_CONTINUATION_COUNT = 10; // safety cap on continuation attempts

/**
 * Creates a new budget tracker for a query loop.
 * One tracker per top-level queryLoop() call.
 */
function createBudgetTracker() {
  return {
    continuationCount: 0,
    lastDeltaTokens: 0,
    lastGlobalTurnTokens: 0,
    startedAt: Date.now(),
  };
}

/**
 * Check whether the query loop should continue or stop based on token budget.
 *
 * @param {Object} tracker - BudgetTracker instance (mutated in place)
 * @param {number|null} budget - Max tokens for this query (null = no budget)
 * @param {number} globalTurnTokens - Total tokens used so far this turn
 * @returns {{ action: 'continue'|'stop', nudgeMessage?: string }}
 */
function checkTokenBudget(tracker, budget, globalTurnTokens) {
  // No budget set — always stop (let the natural loop exit handle things)
  if (budget === null || budget <= 0) {
    return { action: 'stop', reason: null };
  }

  const turnTokens = globalTurnTokens;
  const pct = Math.round((turnTokens / budget) * 100);
  const deltaSinceLastCheck = globalTurnTokens - tracker.lastGlobalTurnTokens;

  // Diminishing returns: if we've run >= 3 continuations and each is producing
  // less than DIMINISHING_THRESHOLD tokens, stop — the model is spinning.
  const isDiminishing =
    tracker.continuationCount >= 3 &&
    deltaSinceLastCheck < DIMINISHING_THRESHOLD &&
    tracker.lastDeltaTokens < DIMINISHING_THRESHOLD;

  // Hit the budget ceiling
  const hitCeiling = turnTokens >= budget * COMPLETION_THRESHOLD;

  // Safety cap
  const hitMaxContinuations = tracker.continuationCount >= MAX_CONTINUATION_COUNT;

  if (isDiminishing || hitCeiling || hitMaxContinuations) {
    return {
      action: 'stop',
      reason: isDiminishing ? 'diminishing_returns' : hitCeiling ? 'budget_ceiling' : 'max_continuations',
      stats: {
        continuationCount: tracker.continuationCount,
        pct,
        turnTokens,
        budget,
        durationMs: Date.now() - tracker.startedAt,
      },
    };
  }

  // Continue — increment tracker
  tracker.continuationCount++;
  tracker.lastDeltaTokens = deltaSinceLastCheck;
  tracker.lastGlobalTurnTokens = globalTurnTokens;

  const nudgeMessage = `[Token budget: ${pct}% used (${turnTokens}/${budget} tokens). ` +
    `Continuation ${tracker.continuationCount}. Be concise and finish quickly.]`;

  return {
    action: 'continue',
    nudgeMessage,
    continuationCount: tracker.continuationCount,
    pct,
  };
}

module.exports = { createBudgetTracker, checkTokenBudget };
```

Wire this into `queryLoop.js`. Create the tracker at the start of `queryLoop()`. After every model call that hits the output limit (Phase 2 recovery path), call `checkTokenBudget(tracker, params.tokenBudget, cumulativeUsage.completion_tokens)`. If `action === 'stop'`, break out of the recovery loop. If `action === 'continue'`, inject the `nudgeMessage` as a system message before the next iteration.

Add `tokenBudget` as an optional parameter to the `queryLoop()` function signature. The `/api/chat` endpoint can pass `tokenBudget: parseInt(req.body.tokenBudget)` if provided by the client.

---

## Appendix J — LSP Integration

**Source**: `src/tools/LSPTool/`, `src/services/lsp/`

The LSP (Language Server Protocol) tool gives the agent IDE-level code intelligence. Instead of asking the model to trace imports manually (which is unreliable and expensive in tokens), the LSP tool calls a running language server to get exact answers. This transforms the agent from a text-manipulator into a code-aware system.

### Capabilities Exposed

The `lsp_query` tool supports these operations, all defined in the `lspToolInputSchema`:

| Operation | What It Does |
|-----------|-------------|
| `hover` | Get type information and documentation for a symbol at a location |
| `go_to_definition` | Find where a symbol is defined |
| `find_references` | Find all usages of a symbol |
| `document_symbols` | List all symbols (functions, classes, vars) in a file |
| `workspace_symbols` | Search for symbols across the entire codebase |
| `prepare_call_hierarchy` | Set up a call hierarchy for a symbol |
| `incoming_calls` | Find what calls a given function |
| `outgoing_calls` | Find what a function calls |

### Architecture

The LSP service (`src/services/lsp/`) maintains a pool of language server processes, one per language. Language servers are launched on demand and kept alive for the session. Supported servers:

- **TypeScript/JavaScript**: `typescript-language-server --stdio`
- **Python**: `pylsp --check-parent-process` or `pyright-langserver --stdio`
- **Go**: `gopls serve`
- **Rust**: `rust-analyzer`

```javascript
// src/services/lsp/manager.js
'use strict';

const { spawn } = require('child_process');
const path = require('path');

// Language → server command mapping
const LSP_SERVERS = {
  typescript: { command: 'typescript-language-server', args: ['--stdio'] },
  javascript: { command: 'typescript-language-server', args: ['--stdio'] },
  python: { command: 'pylsp', args: [] },
  go: { command: 'gopls', args: ['serve'] },
  rust: { command: 'rust-analyzer', args: [] },
};

// Active server instances keyed by language
const servers = new Map();

function detectLanguage(filePath) {
  const ext = path.extname(filePath).toLowerCase();
  const map = {
    '.ts': 'typescript', '.tsx': 'typescript',
    '.js': 'javascript', '.jsx': 'javascript', '.mjs': 'javascript',
    '.py': 'python',
    '.go': 'go',
    '.rs': 'rust',
  };
  return map[ext] || null;
}

async function getLspServer(language, rootUri) {
  if (servers.has(language)) return servers.get(language);

  const config = LSP_SERVERS[language];
  if (!config) return null;

  // Check if the server binary is available
  try {
    require('child_process').execSync(`which ${config.command}`, { stdio: 'ignore' });
  } catch {
    return null; // LSP server not installed — graceful degradation
  }

  const proc = spawn(config.command, config.args, {
    stdio: ['pipe', 'pipe', 'pipe'],
    env: { ...process.env, TSS_LOG: '' },
  });

  // Send LSP initialize request
  const initRequest = JSON.stringify({
    jsonrpc: '2.0', id: 1, method: 'initialize',
    params: {
      rootUri,
      capabilities: {
        textDocument: {
          hover: { dynamicRegistration: false },
          definition: { dynamicRegistration: false },
          references: { dynamicRegistration: false },
          documentSymbol: { dynamicRegistration: false },
        },
      },
    },
  });

  proc.stdin.write(`Content-Length: ${Buffer.byteLength(initRequest)}\r\n\r\n${initRequest}`);

  const server = { proc, language, rootUri, requestId: 2 };
  servers.set(language, server);
  return server;
}

function isLspAvailable(language) {
  const config = LSP_SERVERS[language];
  if (!config) return false;
  try {
    require('child_process').execSync(`which ${config.command}`, { stdio: 'ignore' });
    return true;
  } catch { return false; }
}

module.exports = { getLspServer, detectLanguage, isLspAvailable };
```

```javascript
// src/tools/lsp/lspTool.js
'use strict';

const { getLspServer, detectLanguage, isLspAvailable } = require('../../services/lsp/manager');

class LSPTool extends BaseTool {
  constructor() {
    super();
    this.name = 'lsp_query';
    this.description = 'Query a language server for code intelligence: hover info, go-to-definition, ' +
      'find-references, document symbols, call hierarchy. Requires the appropriate LSP server to be installed.';
    this.inputSchema = {
      type: 'object',
      properties: {
        operation: {
          type: 'string',
          enum: ['hover', 'go_to_definition', 'find_references', 'document_symbols', 'workspace_symbols', 'incoming_calls', 'outgoing_calls'],
        },
        file_path: { type: 'string', description: 'Absolute path to the file.' },
        line: { type: 'integer', description: 'Line number (0-indexed) for position-based operations.' },
        character: { type: 'integer', description: 'Character offset (0-indexed) for position-based operations.' },
        query: { type: 'string', description: 'Symbol name for workspace_symbols.' },
      },
      required: ['operation', 'file_path'],
    };
    this.shouldDefer = true; // Discovered via tool_search
  }

  isReadOnly() { return true; }
  isConcurrencySafe() { return true; }

  async call(input, context) {
    const language = detectLanguage(input.file_path);
    if (!language) {
      return { output: `Unsupported file type: ${path.extname(input.file_path)}` };
    }

    if (!isLspAvailable(language)) {
      return {
        output: `LSP server for ${language} is not installed. ` +
          `Install it with: npm install -g typescript-language-server (TypeScript), ` +
          `pip install python-lsp-server (Python), etc.`,
      };
    }

    const rootUri = `file://${context.cwd}`;
    const server = await getLspServer(language, rootUri);
    if (!server) return { output: 'Failed to start LSP server.' };

    // All LSP operations require sending a request and awaiting a response.
    // The full LSP JSON-RPC request/response cycle is handled by sendLspRequest().
    const fileUri = `file://${input.file_path}`;
    const position = { line: input.line || 0, character: input.character || 0 };

    const methodMap = {
      hover: { method: 'textDocument/hover', params: { textDocument: { uri: fileUri }, position } },
      go_to_definition: { method: 'textDocument/definition', params: { textDocument: { uri: fileUri }, position } },
      find_references: { method: 'textDocument/references', params: { textDocument: { uri: fileUri }, position, context: { includeDeclaration: true } } },
      document_symbols: { method: 'textDocument/documentSymbol', params: { textDocument: { uri: fileUri } } },
      workspace_symbols: { method: 'workspace/symbol', params: { query: input.query || '' } },
    };

    const lspRequest = methodMap[input.operation];
    if (!lspRequest) return { output: `Unknown LSP operation: ${input.operation}` };

    try {
      const result = await sendLspRequest(server, lspRequest.method, lspRequest.params);
      return { output: formatLspResult(input.operation, result) };
    } catch (err) {
      return { output: `LSP error: ${err.message}` };
    }
  }
}

function formatLspResult(operation, result) {
  if (!result) return '(no results)';
  if (operation === 'hover') {
    return typeof result.contents === 'string' ? result.contents : result.contents?.value || '(no hover info)';
  }
  if (operation === 'go_to_definition' || operation === 'find_references') {
    const locations = Array.isArray(result) ? result : [result];
    return locations.map(loc => {
      const uri = loc.uri || loc.targetUri || '';
      const filePath = uri.replace('file://', '');
      const line = (loc.range?.start?.line || loc.targetSelectionRange?.start?.line || 0) + 1;
      return `${filePath}:${line}`;
    }).join('\n');
  }
  if (operation === 'document_symbols' || operation === 'workspace_symbols') {
    const symbols = Array.isArray(result) ? result : [];
    return symbols.slice(0, 50).map(s => `${s.name} (${s.kind})`).join('\n');
  }
  return JSON.stringify(result, null, 2).slice(0, 2000);
}

async function sendLspRequest(server, method, params) {
  return new Promise((resolve, reject) => {
    const id = server.requestId++;
    const request = JSON.stringify({ jsonrpc: '2.0', id, method, params });
    const header = `Content-Length: ${Buffer.byteLength(request)}\r\n\r\n`;

    server.proc.stdin.write(header + request);

    let buffer = '';
    const handler = (chunk) => {
      buffer += chunk.toString();
      const headerEnd = buffer.indexOf('\r\n\r\n');
      if (headerEnd === -1) return;

      const body = buffer.slice(headerEnd + 4);
      try {
        const msg = JSON.parse(body);
        if (msg.id === id) {
          server.proc.stdout.off('data', handler);
          if (msg.error) reject(new Error(msg.error.message));
          else resolve(msg.result);
        }
      } catch { /* incomplete — wait for more data */ }
    };

    server.proc.stdout.on('data', handler);
    setTimeout(() => {
      server.proc.stdout.off('data', handler);
      reject(new Error('LSP request timed out'));
    }, 10_000);
  });
}

module.exports = { LSPTool };
```

---

## Appendix K — Jupyter Notebook Support

**Source**: `src/tools/NotebookEditTool/`

The `NotebookEditTool` in the Claude Code source allows editing Jupyter `.ipynb` files at the cell level — the model can insert, replace, or delete individual cells without touching the rest of the notebook. This is the correct abstraction: notebooks are not plain text files, and treating them as such corrupts their JSON structure.

```javascript
// src/tools/notebook/notebookEdit.js
'use strict';

const fs = require('fs');
const path = require('path');

class NotebookEditTool extends BaseTool {
  constructor() {
    super();
    this.name = 'notebook_edit';
    this.description = 'Edit a Jupyter notebook (.ipynb) at the cell level. ' +
      'Can insert, replace, or delete cells without corrupting the notebook structure.';
    this.inputSchema = {
      type: 'object',
      properties: {
        notebook_path: { type: 'string', description: 'Absolute path to the .ipynb file.' },
        operation: {
          type: 'string',
          enum: ['read', 'insert_cell', 'replace_cell', 'delete_cell', 'append_cell'],
        },
        cell_index: { type: 'integer', description: 'Zero-based cell index (for replace, delete, insert).' },
        cell_type: {
          type: 'string',
          enum: ['code', 'markdown', 'raw'],
          description: 'Cell type (for insert, replace, append).',
          default: 'code',
        },
        source: { type: 'string', description: 'Cell source content (for insert, replace, append).' },
      },
      required: ['notebook_path', 'operation'],
    };
    this.shouldDefer = true;
  }

  isReadOnly(input) { return input?.operation === 'read'; }
  isConcurrencySafe() { return false; }
  isDestructive(input) { return input?.operation === 'delete_cell'; }

  async call(input, context) {
    const notebookPath = path.resolve(context.cwd, input.notebook_path);

    if (!notebookPath.endsWith('.ipynb')) {
      return { output: `Error: ${input.notebook_path} is not a Jupyter notebook (.ipynb).` };
    }

    let notebook;
    try {
      notebook = JSON.parse(fs.readFileSync(notebookPath, 'utf8'));
    } catch (err) {
      return { output: `Error reading notebook: ${err.message}` };
    }

    if (!notebook.cells || !Array.isArray(notebook.cells)) {
      return { output: 'Error: Invalid notebook format — no cells array.' };
    }

    switch (input.operation) {
      case 'read': {
        const summary = notebook.cells.map((cell, i) => {
          const preview = (Array.isArray(cell.source) ? cell.source.join('') : cell.source || '').slice(0, 120);
          return `[${i}] (${cell.cell_type}): ${preview}`;
        }).join('\n');
        return { output: `Notebook: ${notebook.cells.length} cells\n\n${summary}` };
      }

      case 'read_cell': {
        const cell = notebook.cells[input.cell_index];
        if (!cell) return { output: `Cell ${input.cell_index} not found.` };
        const source = Array.isArray(cell.source) ? cell.source.join('') : cell.source;
        return { output: `Cell ${input.cell_index} (${cell.cell_type}):\n${source}` };
      }

      case 'replace_cell': {
        if (input.cell_index < 0 || input.cell_index >= notebook.cells.length) {
          return { output: `Cell index ${input.cell_index} out of range (0-${notebook.cells.length - 1}).` };
        }
        notebook.cells[input.cell_index] = makeCell(input.cell_type || 'code', input.source || '');
        break;
      }

      case 'insert_cell': {
        const idx = Math.min(Math.max(input.cell_index || 0, 0), notebook.cells.length);
        notebook.cells.splice(idx, 0, makeCell(input.cell_type || 'code', input.source || ''));
        break;
      }

      case 'append_cell': {
        notebook.cells.push(makeCell(input.cell_type || 'code', input.source || ''));
        break;
      }

      case 'delete_cell': {
        if (input.cell_index < 0 || input.cell_index >= notebook.cells.length) {
          return { output: `Cell index ${input.cell_index} out of range.` };
        }
        notebook.cells.splice(input.cell_index, 1);
        break;
      }

      default:
        return { output: `Unknown operation: ${input.operation}` };
    }

    // Write the modified notebook back to disk
    fs.writeFileSync(notebookPath, JSON.stringify(notebook, null, 1), 'utf8');
    return { output: `Notebook updated. ${notebook.cells.length} cells total.` };
  }
}

function makeCell(cellType, source) {
  return {
    cell_type: cellType,
    source: source,
    metadata: {},
    outputs: cellType === 'code' ? [] : undefined,
    execution_count: cellType === 'code' ? null : undefined,
  };
}

module.exports = { NotebookEditTool };
```

---

## Appendix L — QueryConfig and Feature Gate Snapshotting

**Source**: `src/query/config.ts`

The Claude Code source separates immutable per-query configuration (snapshotted at the moment `queryLoop()` is called) from mutable per-iteration state. This is the `QueryConfig` pattern. It prevents feature gate checks from happening inside the hot path of the loop (which would re-evaluate the gate on every iteration, potentially with different results) and makes the loop's behavior fully reproducible.

```javascript
// src/utils/queryConfig.js
'use strict';

/**
 * Build an immutable QueryConfig for a single queryLoop invocation.
 * All feature gate checks happen HERE, once, before the loop starts.
 * The loop reads from this config — it never calls feature checks directly.
 *
 * @param {Object} params - Parameters from the API request
 * @returns {Object} Immutable QueryConfig
 */
function buildQueryConfig(params = {}) {
  return Object.freeze({
    sessionId: params.sessionId || require('uuid').v4(),

    // Model configuration
    model: params.model || process.env.EUCLID_MODEL || 'qwen2.5-coder:32b-instruct-q4_K_M',
    compactModel: process.env.EUCLID_COMPACT_MODEL || 'qwen2.5:3b',
    maxTurns: params.maxTurns || parseInt(process.env.EUCLID_MAX_TURNS || '50', 10),
    tokenBudget: params.tokenBudget ? parseInt(params.tokenBudget, 10) : null,

    // Permission mode — cannot change during a query
    permissionMode: params.permissionMode || process.env.EUCLID_PERMISSION_MODE || 'default',

    // Feature flags — evaluated once, not re-evaluated per iteration
    gates: Object.freeze({
      streamingToolExecution: process.env.EUCLID_STREAMING_TOOLS !== 'false',
      emitToolUseSummaries: process.env.EUCLID_EMIT_TOOL_SUMMARIES === 'true',
      extractMemoriesEnabled: process.env.EUCLID_MEMORY_EXTRACT !== 'false',
      hooksEnabled: process.env.EUCLID_HOOKS !== 'false',
      lspEnabled: process.env.EUCLID_LSP === 'true',
      reflexionOnFailure: process.env.EUCLID_REFLEXION !== 'false',
      planModeEnabled: process.env.EUCLID_PLAN_MODE !== 'false',
    }),

    // Runtime info
    cwd: params.cwd || process.cwd(),
    isNonInteractive: params.isNonInteractive || false,
  });
}

module.exports = { buildQueryConfig };
```

Pass `queryConfig` into `queryLoop()` instead of individual parameters. The `QueryEngine.submitMessage()` method calls `buildQueryConfig()` once per submitted message, then passes the frozen config to the loop. This makes it impossible to accidentally mutate feature gate state mid-loop.

Add the following environment variable documentation to `src/server.js` and the production `.env.example`:

```bash
# Feature Gates
EUCLID_STREAMING_TOOLS=true        # Stream tool results as they execute
EUCLID_EMIT_TOOL_SUMMARIES=false   # Emit tool use summary events to SSE
EUCLID_MEMORY_EXTRACT=true         # Auto-extract memories at turn end
EUCLID_HOOKS=true                  # Enable user-defined hooks system
EUCLID_LSP=false                   # Enable LSP tool (requires language servers installed)
EUCLID_REFLEXION=true              # Run Reflexion on tool errors
EUCLID_PLAN_MODE=true              # Enable Plan Mode tools
```

---

## Appendix M — File History and Undo

**Source**: `src/utils/fileHistory.ts` (referenced in `NotebookEditTool`)

The Claude Code source tracks file history — every time a file is edited, a snapshot is stored so the user can undo changes. Euclid implements a lightweight version using the `file_snapshots` table.

```javascript
// src/utils/fileHistory.js
'use strict';

const fs = require('fs');
const crypto = require('crypto');
const { db } = require('../db');

const FILE_HISTORY_ENABLED = process.env.EUCLID_FILE_HISTORY !== 'false';
const MAX_SNAPSHOTS_PER_FILE = 10; // Keep last 10 versions

/**
 * Snapshot a file before editing. Call this in FileWrite and FileEdit tools
 * before writing. This enables undo via /api/files/undo.
 */
function trackFileEdit(filePath, sessionId) {
  if (!FILE_HISTORY_ENABLED) return;

  try {
    const content = fs.readFileSync(filePath, 'utf8');
    const hash = crypto.createHash('sha256').update(content).digest('hex').slice(0, 16);

    // Avoid storing duplicate snapshots (file unchanged)
    const last = db.prepare(`
      SELECT hash FROM file_snapshots
      WHERE file_path = ? ORDER BY created_at DESC LIMIT 1
    `).get(filePath);

    if (last?.hash === hash) return; // same content, skip

    db.prepare(`
      INSERT INTO file_snapshots (file_path, content, hash, session_id, created_at)
      VALUES (?, ?, ?, ?, ?)
    `).run(filePath, content, hash, sessionId || '', Date.now());

    // Prune old snapshots
    db.prepare(`
      DELETE FROM file_snapshots
      WHERE file_path = ? AND id NOT IN (
        SELECT id FROM file_snapshots WHERE file_path = ?
        ORDER BY created_at DESC LIMIT ?
      )
    `).run(filePath, filePath, MAX_SNAPSHOTS_PER_FILE);

  } catch { /* Read failed — file might not exist yet, that's fine */ }
}

/**
 * Restore a file to its previous snapshot. Returns the restored content.
 */
function undoFileEdit(filePath) {
  const snapshot = db.prepare(`
    SELECT content FROM file_snapshots
    WHERE file_path = ? ORDER BY created_at DESC LIMIT 1 OFFSET 1
  `).get(filePath); // OFFSET 1 = skip current (which matches disk), get previous

  if (!snapshot) return null;

  fs.writeFileSync(filePath, snapshot.content, 'utf8');
  db.prepare(`
    DELETE FROM file_snapshots WHERE file_path = ?
    AND created_at = (SELECT created_at FROM file_snapshots WHERE file_path = ? ORDER BY created_at DESC LIMIT 1)
  `).run(filePath, filePath);

  return snapshot.content;
}

module.exports = { trackFileEdit, undoFileEdit };
```

Add the `file_snapshots` table to the schema (it is referenced in Section 2.4 of the main guide but the implementation was missing):

```sql
CREATE TABLE IF NOT EXISTS file_snapshots (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  file_path TEXT NOT NULL,
  content TEXT NOT NULL,
  hash TEXT NOT NULL,
  session_id TEXT DEFAULT '',
  created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);
CREATE INDEX IF NOT EXISTS idx_snapshots_path ON file_snapshots(file_path, created_at DESC);
```

Add `POST /api/files/undo` to the API routes that accepts `{ filePath }` and calls `undoFileEdit()`. Wire `trackFileEdit()` into the `FileWriteTool` and `FileEditTool` implementations just before writing.

---

## Appendix N — MCP Tool Integration

**Source**: `src/tools/MCPTool/`, `src/tools/ListMcpResourcesTool/`, `src/tools/ReadMcpResourceTool/`, `src/services/mcp/`

MCP (Model Context Protocol) is the standard for giving AI agents access to external tools and resources. The Claude Code source has a full MCP connection manager that discovers, connects to, and manages multiple MCP servers simultaneously. Euclid must support MCP for extensibility — any MCP server (database tools, GitHub tools, Slack tools, etc.) can be plugged in without modifying Euclid's core.

### MCP Configuration

MCP servers are configured in `.euclid/mcp.json`:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"],
      "description": "File system access MCP server"
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" },
      "description": "GitHub API MCP server"
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "${DATABASE_URL}"],
      "description": "PostgreSQL MCP server"
    }
  }
}
```

### MCP Client Implementation

```javascript
// src/services/mcp/client.js
'use strict';

const { spawn } = require('child_process');
const fs = require('fs');
const path = require('path');
const readline = require('readline');

// Active MCP server connections keyed by server name
const mcpServers = new Map();

function loadMcpConfig(cwd) {
  const locations = [
    path.join(cwd, '.euclid', 'mcp.json'),
    path.join(process.env.HOME || '', '.euclid', 'mcp.json'),
  ];
  for (const loc of locations) {
    if (fs.existsSync(loc)) {
      try { return JSON.parse(fs.readFileSync(loc, 'utf8')); } catch { }
    }
  }
  return { mcpServers: {} };
}

async function connectMcpServer(name, config, cwd) {
  if (mcpServers.has(name)) return mcpServers.get(name);

  const env = { ...process.env };
  // Expand ${VAR} references in env values
  for (const [k, v] of Object.entries(config.env || {})) {
    env[k] = v.replace(/\$\{([^}]+)\}/g, (_, varName) => process.env[varName] || '');
  }

  const proc = spawn(config.command, config.args || [], {
    stdio: ['pipe', 'pipe', 'pipe'],
    env,
    cwd,
  });

  const server = { proc, name, requestId: 1, pending: new Map() };

  // Wire JSON-RPC response handler
  const rl = readline.createInterface({ input: proc.stdout });
  rl.on('line', (line) => {
    try {
      const msg = JSON.parse(line);
      if (msg.id && server.pending.has(msg.id)) {
        const { resolve, reject } = server.pending.get(msg.id);
        server.pending.delete(msg.id);
        if (msg.error) reject(new Error(msg.error.message));
        else resolve(msg.result);
      }
    } catch { }
  });

  // Initialize
  const initResult = await sendMcpRequest(server, 'initialize', {
    protocolVersion: '2024-11-05',
    capabilities: {},
    clientInfo: { name: 'euclid', version: '1.0.0' },
  });

  await sendMcpRequest(server, 'notifications/initialized', {});

  server.capabilities = initResult?.capabilities || {};
  server.tools = initResult?.capabilities?.tools ? await listMcpTools(server) : [];

  mcpServers.set(name, server);
  return server;
}

async function listMcpTools(server) {
  try {
    const result = await sendMcpRequest(server, 'tools/list', {});
    return result?.tools || [];
  } catch { return []; }
}

async function callMcpTool(serverName, toolName, args) {
  const server = mcpServers.get(serverName);
  if (!server) throw new Error(`MCP server "${serverName}" not connected.`);

  return sendMcpRequest(server, 'tools/call', { name: toolName, arguments: args });
}

function sendMcpRequest(server, method, params) {
  return new Promise((resolve, reject) => {
    const id = server.requestId++;
    const request = JSON.stringify({ jsonrpc: '2.0', id, method, params });

    server.pending.set(id, { resolve, reject });
    server.proc.stdin.write(request + '\n');

    setTimeout(() => {
      if (server.pending.has(id)) {
        server.pending.delete(id);
        reject(new Error(`MCP request timed out: ${method}`));
      }
    }, 30_000);
  });
}

/**
 * Initialize all configured MCP servers at startup.
 * Returns a map of server name → tool list.
 */
async function initializeMcpServers(cwd) {
  const config = loadMcpConfig(cwd);
  const results = {};

  for (const [name, serverConfig] of Object.entries(config.mcpServers || {})) {
    try {
      const server = await connectMcpServer(name, serverConfig, cwd);
      results[name] = server.tools;
      console.log(`[mcp] Connected: ${name} (${server.tools.length} tools)`);
    } catch (err) {
      console.error(`[mcp] Failed to connect ${name}: ${err.message}`);
    }
  }

  return results;
}

/**
 * Build tool definitions for all connected MCP tools.
 * These are added to the model's tool list alongside native Euclid tools.
 */
function getMcpToolDefinitions() {
  const defs = [];
  for (const [serverName, server] of mcpServers) {
    for (const tool of server.tools) {
      defs.push({
        name: `mcp__${serverName}__${tool.name}`,
        description: `[MCP:${serverName}] ${tool.description || ''}`,
        inputSchema: tool.inputSchema || { type: 'object', properties: {} },
        isMcp: true,
        serverName,
        mcpToolName: tool.name,
      });
    }
  }
  return defs;
}

module.exports = { initializeMcpServers, getMcpToolDefinitions, callMcpTool };
```

In `src/tools/runner.js`, detect MCP tools by their `mcp__<server>__<tool>` name prefix and route them to `callMcpTool(serverName, mcpToolName, input)` instead of the native tool runner. Call `initializeMcpServers(cwd)` in `src/server.js` after the database initializes and before the server starts listening.

---

## Appendix O — Expanded Skills Roster (All 29 Skills)

Section 12 of the main guide specifies creating 7 skills. The `everything-claude-code-main.zip` archive contains 29 skills. Euclid must ship all of them. Each skill is a self-contained `.md` file in `skills/<name>/SKILL.md`. The skill selector picks the most relevant ones based on keyword matching.

Update Section 4's scaffold to create all skill directories:

```bash
mkdir -p euclid/skills/{api-design,backend-patterns,coding-standards,e2e-testing,\
frontend-patterns,documentation-lookup,eval-harness,security-review,tdd-workflow,\
verification-loop,mcp-server-patterns,deep-research,strategic-compact,bun-runtime,\
claude-api,exa-search,fal-ai-media,x-api,article-writing,content-engine,crosspost,\
video-editing,frontend-slides,market-research,investor-materials,investor-outreach,\
nextjs-turbopack,dmux-workflows,everything-euclid}
```

The 22 skills beyond the original 7 are described below. For each, create a `SKILL.md` following the frontmatter + description + trigger pattern. The content of each skill is populated from the corresponding file in the zip:

| Skill Name | When to Activate | Key Guidance |
|-----------|-----------------|--------------|
| `security-review` | Auth, input handling, secrets, payment, API endpoints | OWASP checklist, secrets never in code, parameterized queries, rate limiting |
| `tdd-workflow` | Writing new features, fixing bugs, refactoring | Tests FIRST, 80%+ coverage, unit + integration + E2E, red-green-refactor cycle |
| `verification-loop` | After feature completion, before PR, post-refactor | Build → lint → test → type-check → E2E in sequence; STOP if any phase fails |
| `mcp-server-patterns` | Building MCP servers | Tools vs Resources vs Prompts, stdio vs HTTP transport, Zod validation |
| `deep-research` | Research tasks, competitive analysis, due diligence | Multi-source synthesis, citation format, firecrawl/exa patterns |
| `strategic-compact` | Long sessions, multi-phase tasks, context pressure | Compact at milestone boundaries not arbitrary points, preserve decision context |
| `bun-runtime` | Bun-specific projects (adapt to Node.js equivalents for Euclid) | Document Node.js alternatives for all Bun APIs |
| `claude-api` | Integrating Claude/Anthropic APIs | Model selection, token budgets, streaming, tool use schema |
| `exa-search` | Web search integration | Exa API patterns, query formulation, result synthesis |
| `fal-ai-media` | AI image/video generation tasks | fal.ai client patterns, model selection, async polling |
| `x-api` | Twitter/X API integration | OAuth 2.0 flow, rate limits, media upload |
| `article-writing` | Technical article writing | Hook-first structure, code examples, SEO metadata |
| `content-engine` | Bulk content creation | Templates, variation generation, CMS integration |
| `crosspost` | Publishing to multiple platforms | Platform-specific formatting, scheduling |
| `video-editing` | Video processing tasks | ffmpeg patterns, encoding settings |
| `frontend-slides` | Presentation creation | Slide structure, Reveal.js, visual hierarchy |
| `market-research` | Market analysis | Framework selection (TAM/SAM/SOM), data sources |
| `investor-materials` | Pitch decks, one-pagers | Narrative arc, metrics, executive summary |
| `investor-outreach` | Investor email campaigns | Personalization, follow-up cadence |
| `nextjs-turbopack` | Next.js projects | App Router patterns, server components, Turbopack |
| `dmux-workflows` | tmux multi-pane workflows | Session management, pane layouts, synchronization |
| `everything-euclid` | Euclid self-documentation | Meta-skill for documenting Euclid's own patterns |

Update the skill selector in `src/services/skill/selector.js` to recognize all 29 skill names.

---

## Appendix P — Scaffold Corrections

The project scaffold in Section 4 is missing several directories revealed by the source analysis. Run these additional `mkdir` commands after the original scaffold:

```bash
# Additional directories not in Section 4
mkdir -p euclid/{scripts,evals,hooks,worktrees}
mkdir -p euclid/src/tools/{plan,worktree,lsp,notebook,repl,sleep,sendMessage,cron,askUser,toolSearch,webFetch,taskTools}
mkdir -p euclid/src/services/{mcp,hooks,cron,memory}
mkdir -p euclid/src/utils/tokenBudget
mkdir -p euclid/skills/{security-review,tdd-workflow,verification-loop,mcp-server-patterns,deep-research,strategic-compact,bun-runtime,claude-api,exa-search,fal-ai-media,x-api,article-writing,content-engine,crosspost,video-editing,frontend-slides,market-research,investor-materials,investor-outreach,nextjs-turbopack,dmux-workflows,everything-euclid}
```

Additional npm packages required by the new appendices:

```bash
npm install node-cron
# For LSP support (optional — only if EUCLID_LSP=true)
npm install vscode-languageserver-types
# For MCP support
npm install @modelcontextprotocol/sdk
```

Update `package.json` scripts to include:

```json
{
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js",
    "db:reset": "node -e \"require('./src/db').reset()\"",
    "db:migrate": "node src/db/migrate.js",
    "test": "node --test src/**/*.test.js",
    "hooks:validate": "node src/services/hooks/validate.js",
    "mcp:list": "node -e \"require('./src/services/mcp/client').initializeMcpServers(process.cwd()).then(r => console.log(r))\"",
    "memory:clear": "node -e \"require('./src/services/memory/extractMemories').clearAllMemories()\"",
    "evals:run": "node evals/runner.js"
  }
}
```

The updated environment variables file `.env.example` must include all variables from Appendix L in addition to the ones already in Section 19.5.


---

> **ADDITIONS VOLUME II — Full src.zip subsystem analysis.**
> Every directory from the 1,902-file Claude Code source has been analyzed. What follows covers every major subsystem not yet specified: the bridge/remote-control layer, swarm multi-agent backends, voice mode, computer use, AutoDream consolidation, streaming tool execution, plugin architecture, bundled skills, permission system depth, compact variants, memory depth, context analysis, hooks types, session backgrounding, teleport, cost tracking, effort scoring, sandbox, rate-limit handling, and every missing tool. These are additive appendices — nothing above is changed.

---

## Appendix Q — Remote Bridge and Live Session Sharing

**Source**: `src/bridge/` (31 files) — `bridgeApi.ts`, `bridgeMain.ts`, `bridgeMessaging.ts`, `bridgePermissionCallbacks.ts`, `replBridge.ts`, `replBridgeTransport.ts`, `jwtUtils.ts`, `sessionRunner.ts`, `trustedDevice.ts`, `workSecret.ts`, `flushGate.ts`, `inboundMessages.ts`, `capacityWake.ts`

The bridge system in Claude Code allows a remote operator (e.g. a web browser or mobile client) to control a running CLI session over WebSocket or SSE. Euclid adapts this into a **live session sharing** feature: a running Euclid session can be exposed to a second client over a signed WebSocket channel, enabling collaborative or remote-controlled coding sessions.

### Architecture

```
Browser A (primary)  ──────►  Euclid Express Server  ──────►  Ollama
                                        │
Browser B (remote)   ◄──────  /bridge/:sessionId (WS)
```

The bridge has three layers: **authentication** (JWT signed with the session secret), **message routing** (inbound from remote, outbound to Euclid's SSE stream), and **permission callbacks** (the remote client can approve or deny tool calls that require human confirmation).

### JWT Session Tokens

```javascript
// src/services/bridge/jwtUtils.js
'use strict';

const crypto = require('crypto');

// Session secrets are stored in the `secrets` table, keyed by sessionId.
// They are generated once at session creation and never change.
function generateSessionSecret() {
  return crypto.randomBytes(32).toString('hex');
}

// Sign a bridge token. The payload is: { sessionId, role, iat, exp }
// role is 'primary' (full control) or 'observer' (read-only mirror).
function signBridgeToken(sessionId, secret, role = 'observer', ttlSeconds = 3600) {
  const header = Buffer.from(JSON.stringify({ alg: 'HS256', typ: 'JWT' })).toString('base64url');
  const payload = Buffer.from(JSON.stringify({
    sessionId,
    role,
    iat: Math.floor(Date.now() / 1000),
    exp: Math.floor(Date.now() / 1000) + ttlSeconds,
  })).toString('base64url');

  const signature = crypto
    .createHmac('sha256', secret)
    .update(`${header}.${payload}`)
    .digest('base64url');

  return `${header}.${payload}.${signature}`;
}

function verifyBridgeToken(token, secret) {
  const [header, payload, sig] = token.split('.');
  if (!header || !payload || !sig) throw new Error('Malformed bridge token.');

  const expectedSig = crypto
    .createHmac('sha256', secret)
    .update(`${header}.${payload}`)
    .digest('base64url');

  if (!crypto.timingSafeEqual(Buffer.from(sig), Buffer.from(expectedSig))) {
    throw new Error('Bridge token signature invalid.');
  }

  const data = JSON.parse(Buffer.from(payload, 'base64url').toString());
  if (data.exp < Math.floor(Date.now() / 1000)) throw new Error('Bridge token expired.');

  return data;
}

module.exports = { generateSessionSecret, signBridgeToken, verifyBridgeToken };
```

### WebSocket Bridge Server

```javascript
// src/services/bridge/bridgeServer.js
'use strict';

const { WebSocketServer } = require('ws');
const { verifyBridgeToken } = require('./jwtUtils');
const { db } = require('../../db');

// Active bridge connections: sessionId → Set<WebSocket>
const bridgeClients = new Map();

// Active session SSE emitters: sessionId → EventEmitter
// Set by the /api/chat route when a session starts streaming.
const sessionEmitters = new Map();

function attachBridgeServer(httpServer) {
  const wss = new WebSocketServer({ server: httpServer, path: '/bridge' });

  wss.on('connection', (ws, req) => {
    const url = new URL(req.url, `ws://${req.headers.host}`);
    const token = url.searchParams.get('token');
    const sessionId = url.searchParams.get('session');

    if (!token || !sessionId) {
      ws.close(4001, 'Missing token or session.');
      return;
    }

    // Look up session secret
    const session = db.prepare('SELECT secret FROM sessions WHERE id = ?').get(sessionId);
    if (!session) { ws.close(4004, 'Session not found.'); return; }

    let bridgeInfo;
    try {
      bridgeInfo = verifyBridgeToken(token, session.secret);
    } catch (err) {
      ws.close(4003, err.message);
      return;
    }

    if (!bridgeClients.has(sessionId)) bridgeClients.set(sessionId, new Set());
    bridgeClients.get(sessionId).add(ws);

    // Mirror all SSE events from the session to this WebSocket client
    const emitter = sessionEmitters.get(sessionId);
    if (emitter) {
      emitter.on('event', (event) => {
        if (ws.readyState === ws.OPEN) ws.send(JSON.stringify(event));
      });
    }

    // Primary role: accept tool permission approvals and messages from the remote
    if (bridgeInfo.role === 'primary') {
      ws.on('message', (raw) => {
        try {
          const msg = JSON.parse(raw.toString());
          handleBridgeInbound(sessionId, msg);
        } catch { /* ignore malformed */ }
      });
    }

    ws.on('close', () => {
      bridgeClients.get(sessionId)?.delete(ws);
    });

    ws.send(JSON.stringify({ type: 'bridge_connected', role: bridgeInfo.role, sessionId }));
  });

  return wss;
}

// Broadcast a session event to all bridge clients watching that session
function broadcastToSession(sessionId, event) {
  const clients = bridgeClients.get(sessionId);
  if (!clients) return;
  const payload = JSON.stringify(event);
  for (const ws of clients) {
    if (ws.readyState === ws.OPEN) ws.send(payload);
  }
}

// Handle inbound messages from a primary bridge client
function handleBridgeInbound(sessionId, msg) {
  if (msg.type === 'tool_approval') {
    // Inject a tool approval into the waiting permission queue
    const queue = pendingPermissions.get(`${sessionId}:${msg.toolUseId}`);
    if (queue) queue.resolve(msg.approved);
  }
  if (msg.type === 'user_message') {
    // Inject a user message into the session's message queue
    const q = sessionMessageQueues.get(sessionId);
    if (q) q.push(msg.content);
  }
}

// Pending permission callbacks: `${sessionId}:${toolUseId}` → { resolve, reject }
const pendingPermissions = new Map();
const sessionMessageQueues = new Map();

// Register a session's SSE emitter so bridge clients can mirror it
function registerSessionEmitter(sessionId, emitter) {
  sessionEmitters.set(sessionId, emitter);
}

module.exports = {
  attachBridgeServer,
  broadcastToSession,
  registerSessionEmitter,
  pendingPermissions,
  sessionMessageQueues,
};
```

### Bridge API Routes

```javascript
// src/routes/bridge.js — Add to server.js
router.post('/api/bridge/create', auth, (req, res) => {
  const { sessionId, role = 'observer', ttlSeconds = 3600 } = req.body;

  const session = db.prepare('SELECT id, secret FROM sessions WHERE id = ?').get(sessionId);
  if (!session) return res.status(404).json({ error: 'Session not found' });

  // Generate a secret if session doesn't have one yet
  if (!session.secret) {
    const secret = generateSessionSecret();
    db.prepare('UPDATE sessions SET secret = ? WHERE id = ?').run(secret, sessionId);
    session.secret = secret;
  }

  const token = signBridgeToken(sessionId, session.secret, role, ttlSeconds);
  const bridgeUrl = `${process.env.BASE_URL || 'ws://localhost:3000'}/bridge?session=${sessionId}&token=${token}`;

  res.json({ token, bridgeUrl, role, expiresIn: ttlSeconds });
});

router.get('/api/bridge/sessions/:sessionId/status', auth, (req, res) => {
  const clients = bridgeClients.get(req.params.sessionId);
  res.json({
    sessionId: req.params.sessionId,
    connectedClients: clients?.size || 0,
  });
});
```

Add `sessions.secret TEXT DEFAULT NULL` to the sessions table. Install `npm install ws`. Add `attachBridgeServer(httpServer)` in `src/server.js` after `const httpServer = http.createServer(app)`.

---

## Appendix R — Swarm System: Multi-Agent Teams with Terminal Backends

**Source**: `src/utils/swarm/` (20 files) — `backends/TmuxBackend.ts`, `backends/ITermBackend.ts`, `backends/InProcessBackend.ts`, `backends/detection.ts`, `teammateInit.ts`, `teammateLayoutManager.ts`, `spawnInProcess.ts`, `permissionSync.ts`, `leaderPermissionBridge.ts`, `teamHelpers.ts`, `constants.ts`, `reconnection.ts`

The swarm system is one of the most powerful features in Claude Code. It spawns multiple agent instances running in parallel in terminal panes (tmux or iTerm2), each working on a different subtask, with a **leader agent** coordinating them and a **permission bridge** routing approval requests back to a single human operator. Euclid adapts this for a backend server context: swarm agents run as child processes or in-process workers, coordinated through the message queue.

### Swarm Backends

Three backends exist in the source. Euclid implements all three:

**Backend 1: In-Process** — Agents run in the same Node.js process as worker threads. Lowest overhead, shared memory, but no visual terminal isolation.

**Backend 2: Tmux** — Each agent gets its own tmux pane. Requires tmux installed. Best for server deployments where you want to watch agents work in split panes.

**Backend 3: iTerm2** — macOS only. Each agent gets a dedicated iTerm2 split pane with its own profile. Best for local development with visual fidelity.

```javascript
// src/services/swarm/backends/detection.js
'use strict';

const { execSync } = require('child_process');

function detectSwarmBackend() {
  // Check in order: iTerm2 (macOS), tmux (any), in-process (fallback)
  if (process.env.TERM_PROGRAM === 'iTerm.app' && process.platform === 'darwin') {
    return 'iterm2';
  }

  try {
    execSync('tmux info', { stdio: 'ignore' });
    // Also check if we're already inside a tmux session
    if (process.env.TMUX) return 'tmux';
  } catch { /* tmux not available */ }

  // Always available as fallback
  return 'in_process';
}

// Swarm constants matching Claude Code's constants.ts
const SWARM_CONSTANTS = {
  MAX_TEAMMATES: 8,           // Hard limit on parallel agents
  PERMISSION_SYNC_INTERVAL: 500, // ms — how often to poll for permission decisions
  TEAMMATE_IDLE_TIMEOUT: 300_000, // 5 min — kill idle agents
  STATUS_POLL_INTERVAL: 2000,    // How often to check agent status
  LEADER_HEARTBEAT_INTERVAL: 10_000, // How often leader pings teammates
};

module.exports = { detectSwarmBackend, SWARM_CONSTANTS };
```

```javascript
// src/services/swarm/backends/tmux.js
'use strict';

const { execSync, spawn } = require('child_process');
const { v4: uuidv4 } = require('uuid');

class TmuxBackend {
  constructor(options = {}) {
    this.sessionName = options.sessionName || `euclid-swarm-${Date.now()}`;
    this.agents = new Map(); // agentId → { pane, process, status }
  }

  /**
   * Create a new tmux session for this swarm if one doesn't exist.
   */
  init() {
    try {
      execSync(`tmux new-session -d -s "${this.sessionName}" -x 220 -y 50`, { stdio: 'ignore' });
    } catch {
      // Session may already exist — that's fine
    }
  }

  /**
   * Spawn a new agent in a tmux split pane.
   * Returns the agentId for tracking.
   */
  spawnAgent(agentId, { command, cwd, env = {}, title }) {
    this.init();

    // Create a horizontal split
    const paneId = execSync(
      `tmux split-window -t "${this.sessionName}" -h -d -P -F "#{pane_id}" -c "${cwd}"`,
      { encoding: 'utf8' }
    ).trim();

    // Set the pane title
    if (title) {
      execSync(`tmux select-pane -t "${paneId}" -T "${title}"`);
    }

    // Build the env prefix and start the command
    const envStr = Object.entries(env).map(([k, v]) => `${k}=${JSON.stringify(v)}`).join(' ');
    execSync(`tmux send-keys -t "${paneId}" "${envStr} ${command}" Enter`);

    this.agents.set(agentId, { paneId, status: 'running', title });

    return agentId;
  }

  /**
   * Send a message/command to a running agent's pane.
   */
  sendInput(agentId, text) {
    const agent = this.agents.get(agentId);
    if (!agent) throw new Error(`Agent ${agentId} not found.`);
    execSync(`tmux send-keys -t "${agent.paneId}" "${text}" Enter`);
  }

  /**
   * Kill a specific agent's pane.
   */
  killAgent(agentId) {
    const agent = this.agents.get(agentId);
    if (!agent) return;
    try { execSync(`tmux kill-pane -t "${agent.paneId}"`); } catch { }
    this.agents.delete(agentId);
  }

  /**
   * Kill the entire swarm session.
   */
  killAll() {
    try { execSync(`tmux kill-session -t "${this.sessionName}"`); } catch { }
    this.agents.clear();
  }

  /**
   * List all panes and their statuses.
   */
  status() {
    return [...this.agents.entries()].map(([id, info]) => ({ id, ...info }));
  }
}

module.exports = { TmuxBackend };
```

```javascript
// src/services/swarm/backends/inProcess.js
'use strict';

const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');
const path = require('path');
const EventEmitter = require('events');

class InProcessBackend extends EventEmitter {
  constructor() {
    super();
    this.agents = new Map(); // agentId → { worker, status, output[] }
  }

  spawnAgent(agentId, { sessionId, systemPrompt, tools, model, initialMessage }) {
    // Each in-process agent is a Worker thread running a stripped-down queryLoop
    const workerScript = path.join(__dirname, '../../workers/agentWorker.js');

    const worker = new Worker(workerScript, {
      workerData: {
        agentId,
        sessionId,
        systemPrompt,
        model,
        initialMessage,
      },
    });

    const agentState = { worker, status: 'running', output: [], agentId };
    this.agents.set(agentId, agentState);

    worker.on('message', (msg) => {
      agentState.output.push(msg);
      this.emit('agent_message', { agentId, ...msg });
    });

    worker.on('exit', (code) => {
      agentState.status = code === 0 ? 'completed' : 'failed';
      agentState.exitCode = code;
      this.emit('agent_done', { agentId, exitCode: code });
    });

    worker.on('error', (err) => {
      agentState.status = 'error';
      agentState.error = err.message;
      this.emit('agent_error', { agentId, error: err.message });
    });

    return agentId;
  }

  killAgent(agentId) {
    const agent = this.agents.get(agentId);
    if (!agent) return;
    agent.worker.terminate();
    agent.status = 'killed';
  }

  killAll() {
    for (const [id] of this.agents) this.killAgent(id);
  }

  getOutput(agentId) {
    return this.agents.get(agentId)?.output || [];
  }

  status() {
    return [...this.agents.entries()].map(([id, a]) => ({
      id,
      status: a.status,
      outputLines: a.output.length,
    }));
  }
}

module.exports = { InProcessBackend };
```

### Agent Worker Thread

```javascript
// src/workers/agentWorker.js — runs inside Worker threads for in-process swarm
'use strict';

const { parentPort, workerData } = require('worker_threads');
const { queryLoop } = require('../services/queryLoop');
const { buildSystemPrompt } = require('../utils/systemPrompt');
const { buildQueryConfig } = require('../utils/queryConfig');
const { loadTools } = require('../tools');

async function main() {
  const { agentId, sessionId, systemPrompt, model, initialMessage } = workerData;

  const config = buildQueryConfig({ sessionId, model });
  const tools = loadTools({ sessionId, permissionMode: 'agent', cwd: process.cwd() });

  const messages = [{ role: 'user', content: initialMessage }];

  const ac = new AbortController();

  try {
    for await (const event of queryLoop({
      messages,
      systemPrompt,
      tools,
      toolContext: { sessionId, agentId, permissionMode: 'agent', cwd: process.cwd() },
      signal: ac.signal,
      queryConfig: config,
    })) {
      parentPort.postMessage(event);
    }
  } catch (err) {
    parentPort.postMessage({ type: 'error', error: err.message });
  }
}

main();
```

### Swarm Orchestrator

```javascript
// src/services/swarm/swarmOrchestrator.js
'use strict';

const { detectSwarmBackend, SWARM_CONSTANTS } = require('./backends/detection');
const { TmuxBackend } = require('./backends/tmux');
const { InProcessBackend } = require('./backends/inProcess');
const { db } = require('../../db');
const { v4: uuidv4 } = require('uuid');
const { complete } = require('../ollama/client');
const { getModel } = require('../../utils/modelRouter');

// Active swarms: swarmId → { backend, leaderAgentId, teammates[], permissionBridge }
const activeSwarms = new Map();

/**
 * Create a new swarm for a task. The leader decomposes the task into subtasks,
 * each assigned to a specialist teammate running in parallel.
 */
async function createSwarm({ task, sessionId, maxTeammates = 4, backendOverride }) {
  const swarmId = uuidv4();
  const backend = backendOverride || detectSwarmBackend();

  const BackendClass = {
    tmux: TmuxBackend,
    in_process: InProcessBackend,
  }[backend] || InProcessBackend;

  const backendInstance = new BackendClass({ sessionName: `euclid-${swarmId.slice(0, 8)}` });

  // Step 1: Leader decomposes the task into parallel subtasks
  const decomposition = await decomposeTask(task, maxTeammates);

  const teammates = [];

  // Step 2: Spawn a teammate for each subtask
  for (const subtask of decomposition.subtasks) {
    const agentId = `agent-${uuidv4().slice(0, 8)}`;
    const systemPrompt = buildTeammateSystemPrompt(subtask, swarmId, agentId);

    backendInstance.spawnAgent(agentId, {
      sessionId,
      systemPrompt,
      model: getModel('main'),
      initialMessage: subtask.description,
      title: subtask.title,
    });

    teammates.push({
      agentId,
      subtask,
      status: 'running',
      output: [],
    });

    // Record in DB
    db.prepare(`
      INSERT INTO swarm_agents (id, swarm_id, session_id, subtask, status, created_at)
      VALUES (?, ?, ?, ?, 'running', ?)
    `).run(agentId, swarmId, sessionId, JSON.stringify(subtask), Date.now());
  }

  activeSwarms.set(swarmId, { swarmId, backend: backendInstance, teammates, sessionId, task });

  // Step 3: Set up permission bridge — all teammate permission requests route to the leader
  if (backendInstance instanceof InProcessBackend) {
    backendInstance.on('agent_message', ({ agentId, type, ...rest }) => {
      if (type === 'permission_required') {
        handleTeammatePermissionRequest(swarmId, agentId, rest);
      }
    });
  }

  return { swarmId, teammates, backend };
}

/**
 * Use the Planner model to decompose a task into parallel subtasks.
 */
async function decomposeTask(task, maxSubtasks) {
  const result = await complete({
    model: getModel('planner'),
    messages: [{
      role: 'user',
      content: `Decompose this task into ${maxSubtasks} parallel subtasks that can be worked on simultaneously without conflicts.

Task: ${task}

Output ONLY valid JSON:
{
  "subtasks": [
    { "id": "1", "title": "Short title", "description": "Full task description", "specialist": "coder|researcher|writer|analyst" },
    ...
  ]
}`,
    }],
    options: { temperature: 0.2, num_predict: 2048 },
  });

  try {
    const content = result?.message?.content || '';
    const json = content.replace(/```json\s*/g, '').replace(/```/g, '').trim();
    return JSON.parse(json);
  } catch {
    // Fallback: treat as single sequential task
    return { subtasks: [{ id: '1', title: task.slice(0, 50), description: task, specialist: 'coder' }] };
  }
}

/**
 * Handle a permission request bubbled up from a teammate.
 * The permission bridge shows it to the human operator via SSE.
 */
function handleTeammatePermissionRequest(swarmId, agentId, request) {
  const swarm = activeSwarms.get(swarmId);
  if (!swarm) return;

  // Emit to the session's SSE stream for human review
  const { broadcastToSession } = require('../bridge/bridgeServer');
  broadcastToSession(swarm.sessionId, {
    type: 'swarm_permission_request',
    swarmId,
    agentId,
    ...request,
  });
}

/**
 * Build the system prompt for a swarm teammate.
 */
function buildTeammateSystemPrompt(subtask, swarmId, agentId) {
  return `You are a specialist agent in a multi-agent swarm.

Swarm ID: ${swarmId}
Your Agent ID: ${agentId}
Your Role: ${subtask.specialist || 'coder'}
Your Task: ${subtask.description}

Rules:
1. Focus only on your assigned task. Do not attempt other subtasks.
2. When you finish, output: [TASK COMPLETE: ${agentId}] followed by a summary.
3. If you need help from another agent, output: [SEND_MESSAGE: <agent_id> | <message>]
4. If blocked waiting on resources, use the sleep tool and retry.
5. Minimize tool calls. Be direct and efficient.`;
}

module.exports = { createSwarm, activeSwarms };
```

Add the `swarm_agents` table to the schema:

```sql
CREATE TABLE IF NOT EXISTS swarm_agents (
  id TEXT PRIMARY KEY,
  swarm_id TEXT NOT NULL,
  session_id TEXT NOT NULL,
  subtask TEXT DEFAULT '{}',
  status TEXT DEFAULT 'pending' CHECK(status IN ('pending','running','completed','failed','killed')),
  output TEXT DEFAULT '',
  created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000),
  completed_at INTEGER
);
CREATE INDEX IF NOT EXISTS idx_swarm_agents_swarm ON swarm_agents(swarm_id, status);
```

Add `POST /api/swarm/create`, `GET /api/swarm/:id/status`, `POST /api/swarm/:id/kill` to `src/routes/agents.js`. Install `npm install ws` for the bridge server WebSocket support.

---

## Appendix S — Voice Mode Integration

**Source**: `src/voice/voiceModeEnabled.ts`, `src/services/voice.ts`, `src/services/voiceStreamSTT.ts`, `src/services/voiceKeyterms.ts`, `src/context/voice.tsx`

Voice mode lets users speak to Euclid instead of typing. The system uses either OpenAI Whisper (local via `whisper.cpp`) or a cloud STT provider. The transcription is injected into the chat loop as a regular user message. The key insight from `voiceStreamSTT.ts` is that streaming STT is used — transcription happens incrementally as the user speaks, not just when they stop, which gives faster perceived latency.

### Voice Architecture

```
Microphone → whisper.cpp (local)  → text → /api/voice/transcribe → chat loop
         OR
Microphone → browser MediaRecorder → /api/voice/upload → whisper.cpp → text
```

```javascript
// src/services/voice/stt.js
'use strict';

const { execFile } = require('child_process');
const fs = require('fs');
const path = require('path');
const os = require('os');
const { v4: uuidv4 } = require('uuid');

// Supported backends in order of preference
const STT_BACKENDS = ['whisper_local', 'openai_whisper', 'none'];

/**
 * Detect which STT backend is available.
 */
function detectSTTBackend() {
  // Check for local whisper.cpp
  try {
    require('child_process').execSync('which whisper', { stdio: 'ignore' });
    return 'whisper_local';
  } catch { }

  // Check for OpenAI API key
  if (process.env.OPENAI_API_KEY) return 'openai_whisper';

  return 'none';
}

const STT_BACKEND = process.env.EUCLID_STT_BACKEND || detectSTTBackend();

/**
 * Transcribe audio data (Buffer, WAV or MP3) to text.
 */
async function transcribe(audioBuffer, { language = 'en', model = 'base' } = {}) {
  if (STT_BACKEND === 'none') {
    throw new Error('No STT backend available. Install whisper.cpp or set OPENAI_API_KEY.');
  }

  // Write audio to temp file
  const tmpFile = path.join(os.tmpdir(), `euclid-voice-${uuidv4()}.wav`);
  fs.writeFileSync(tmpFile, audioBuffer);

  try {
    if (STT_BACKEND === 'whisper_local') {
      return await transcribeLocal(tmpFile, { language, model });
    }
    if (STT_BACKEND === 'openai_whisper') {
      return await transcribeOpenAI(tmpFile, { language });
    }
  } finally {
    fs.unlinkSync(tmpFile);
  }
}

async function transcribeLocal(audioPath, { language, model }) {
  return new Promise((resolve, reject) => {
    execFile('whisper', [
      audioPath,
      '--language', language,
      '--model', model,
      '--output-txt',
      '--output-file', audioPath,
    ], (err, stdout) => {
      if (err) { reject(err); return; }
      // whisper outputs to <file>.txt
      try {
        const text = fs.readFileSync(`${audioPath}.txt`, 'utf8').trim();
        fs.unlinkSync(`${audioPath}.txt`);
        resolve(text);
      } catch { resolve(stdout.trim()); }
    });
  });
}

async function transcribeOpenAI(audioPath, { language }) {
  const FormData = require('form-data');
  const form = new FormData();
  form.append('file', fs.createReadStream(audioPath));
  form.append('model', 'whisper-1');
  form.append('language', language);

  const res = await fetch('https://api.openai.com/v1/audio/transcriptions', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
      ...form.getHeaders(),
    },
    body: form,
  });

  const data = await res.json();
  return data.text || '';
}

/**
 * Voice key-terms extraction: after transcription, extract intent keywords
 * that help the skill selector pick relevant skills.
 * Source: src/services/voiceKeyterms.ts
 */
function extractVoiceKeyterms(transcript) {
  // Simple approach: extract nouns and verbs after stopwords removal
  const STOP_WORDS = new Set(['the', 'a', 'an', 'is', 'are', 'was', 'can', 'you', 'i', 'me', 'my', 'please', 'to', 'and', 'or', 'for', 'in', 'on', 'at']);
  return transcript.toLowerCase()
    .replace(/[^a-z\s]/g, '')
    .split(/\s+/)
    .filter(w => w.length > 2 && !STOP_WORDS.has(w))
    .slice(0, 10);
}

module.exports = { transcribe, detectSTTBackend, extractVoiceKeyterms, STT_BACKEND };
```

```javascript
// src/routes/voice.js
'use strict';

const router = require('express').Router();
const multer = require('multer');
const { transcribe, STT_BACKEND, extractVoiceKeyterms } = require('../services/voice/stt');

const upload = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: 25 * 1024 * 1024 }, // 25MB — OpenAI Whisper limit
  fileFilter: (req, file, cb) => {
    const allowed = ['audio/wav', 'audio/mpeg', 'audio/webm', 'audio/ogg', 'audio/mp4'];
    cb(null, allowed.includes(file.mimetype));
  },
});

// POST /api/voice/transcribe — accept audio upload, return text
router.post('/transcribe', upload.single('audio'), async (req, res) => {
  if (STT_BACKEND === 'none') {
    return res.status(503).json({
      error: 'Voice mode unavailable. Install whisper.cpp or set OPENAI_API_KEY.',
      backend: 'none',
    });
  }

  if (!req.file) {
    return res.status(400).json({ error: 'No audio file provided.' });
  }

  try {
    const text = await transcribe(req.file.buffer, {
      language: req.body.language || 'en',
      model: req.body.model || 'base',
    });

    const keyterms = extractVoiceKeyterms(text);

    res.json({ text, keyterms, backend: STT_BACKEND });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// GET /api/voice/status — check voice availability
router.get('/status', (req, res) => {
  res.json({
    available: STT_BACKEND !== 'none',
    backend: STT_BACKEND,
    languages: ['en', 'es', 'fr', 'de', 'it', 'pt', 'zh', 'ja', 'ko', 'ar'],
  });
});

module.exports = router;
```

Add `app.use('/api/voice', voiceRouter)` to `src/server.js`. Add `npm install form-data` for the OpenAI path. Add `EUCLID_STT_BACKEND=whisper_local` and `OPENAI_API_KEY=` to `.env.example`.

---

## Appendix T — Computer Use (Desktop Automation)

**Source**: `src/utils/computerUse/` (12 files) — `executor.ts`, `hostAdapter.ts`, `mcpServer.ts`, `setup.ts`, `computerUseLock.ts`, `escHotkey.ts`, `gates.ts`, `drainRunLoop.ts`, `cleanup.ts`

Computer Use gives the agent the ability to control the desktop: take screenshots, move the mouse, click, type, and interact with GUI applications. In the Claude Code source, this is implemented as an MCP server that wraps native OS automation. Euclid adapts this for server-side deployments using `xdotool` (Linux) and `AppleScript` (macOS).

The **computer use lock** from `computerUseLock.ts` ensures only one computer-use operation runs at a time — parallel GUI operations cause race conditions. The **ESC hotkey** from `escHotkey.ts` gives the user an emergency abort (pressing ESC 3 times rapidly cancels any running computer-use action).

```javascript
// src/services/computerUse/executor.js
'use strict';

const { execSync, execFile } = require('child_process');
const fs = require('fs');
const os = require('os');
const path = require('path');

// Mutex: only one computer-use action at a time
let lockHolder = null;
const lockQueue = [];

async function acquireComputerUseLock(agentId) {
  if (!lockHolder) {
    lockHolder = agentId;
    return () => { lockHolder = null; drainLockQueue(); };
  }
  return new Promise((resolve) => {
    lockQueue.push({ agentId, resolve });
  });
}

function drainLockQueue() {
  if (!lockQueue.length) return;
  const next = lockQueue.shift();
  lockHolder = next.agentId;
  next.resolve(() => { lockHolder = null; drainLockQueue(); });
}

// Platform detection
const PLATFORM = process.platform;
const HAS_XDOTOOL = (() => {
  try { execSync('which xdotool', { stdio: 'ignore' }); return true; } catch { return false; }
})();

/**
 * Take a screenshot. Returns base64 PNG.
 */
async function screenshot(region) {
  const tmpFile = path.join(os.tmpdir(), `euclid-screenshot-${Date.now()}.png`);
  try {
    if (PLATFORM === 'linux' && HAS_XDOTOOL) {
      execSync(`scrot "${tmpFile}"`, { stdio: 'ignore' });
    } else if (PLATFORM === 'darwin') {
      execSync(`screencapture -x "${tmpFile}"`, { stdio: 'ignore' });
    } else {
      throw new Error('Screenshots not supported on this platform.');
    }
    const buf = fs.readFileSync(tmpFile);
    return buf.toString('base64');
  } finally {
    if (fs.existsSync(tmpFile)) fs.unlinkSync(tmpFile);
  }
}

/**
 * Move mouse to absolute coordinates.
 */
async function moveMouse(x, y) {
  if (PLATFORM === 'linux' && HAS_XDOTOOL) {
    execSync(`xdotool mousemove ${x} ${y}`);
  } else if (PLATFORM === 'darwin') {
    execSync(`osascript -e 'tell application "System Events" to set position of mouse to {${x}, ${y}}'`);
  }
}

/**
 * Click mouse button. button: 1=left, 2=middle, 3=right
 */
async function click(x, y, button = 1) {
  if (PLATFORM === 'linux' && HAS_XDOTOOL) {
    execSync(`xdotool mousemove ${x} ${y} click ${button}`);
  } else if (PLATFORM === 'darwin') {
    execSync(`osascript -e 'tell application "System Events" to click at {${x}, ${y}}'`);
  }
}

/**
 * Type text at the current focus.
 */
async function typeText(text) {
  const escaped = text.replace(/"/g, '\\"');
  if (PLATFORM === 'linux' && HAS_XDOTOOL) {
    execSync(`xdotool type --clearmodifiers "${escaped}"`);
  } else if (PLATFORM === 'darwin') {
    execSync(`osascript -e 'tell application "System Events" to keystroke "${escaped}"'`);
  }
}

/**
 * Press a key or key combination. e.g. "ctrl+c", "Return", "Tab"
 */
async function pressKey(key) {
  const normalized = key.toLowerCase().replace(/\+/g, '+');
  if (PLATFORM === 'linux' && HAS_XDOTOOL) {
    execSync(`xdotool key ${normalized}`);
  } else if (PLATFORM === 'darwin') {
    execSync(`osascript -e 'tell application "System Events" to key code ${keyToCode(key)}'`);
  }
}

/**
 * Scroll at the given position.
 */
async function scroll(x, y, direction = 'down', amount = 3) {
  const button = direction === 'down' ? 5 : 4;
  if (PLATFORM === 'linux' && HAS_XDOTOOL) {
    for (let i = 0; i < amount; i++) {
      execSync(`xdotool mousemove ${x} ${y} click ${button}`);
    }
  }
}

function keyToCode(key) {
  // Simplified macOS key code map
  const codes = { Return: 36, Tab: 48, Escape: 53, space: 49, BackSpace: 51 };
  return codes[key] || 0;
}

module.exports = {
  acquireComputerUseLock,
  screenshot,
  moveMouse,
  click,
  typeText,
  pressKey,
  scroll,
};
```

```javascript
// src/tools/computerUse/computerUseTool.js
'use strict';

const {
  acquireComputerUseLock,
  screenshot,
  moveMouse,
  click,
  typeText,
  pressKey,
  scroll,
} = require('../../services/computerUse/executor');

class ComputerUseTool extends BaseTool {
  constructor() {
    super();
    this.name = 'computer_use';
    this.description = 'Control the desktop: take screenshots, move mouse, click, type, scroll. ' +
      'Use for GUI automation when no CLI alternative exists. ' +
      'ALWAYS take a screenshot first to confirm the screen state before clicking.';
    this.inputSchema = {
      type: 'object',
      properties: {
        action: {
          type: 'string',
          enum: ['screenshot', 'move_mouse', 'click', 'double_click', 'right_click', 'type', 'key', 'scroll'],
        },
        x: { type: 'integer', description: 'X coordinate (for mouse actions).' },
        y: { type: 'integer', description: 'Y coordinate (for mouse actions).' },
        text: { type: 'string', description: 'Text to type or key to press.' },
        direction: { type: 'string', enum: ['up', 'down', 'left', 'right'], default: 'down' },
        amount: { type: 'integer', description: 'Scroll amount (default 3).', default: 3 },
      },
      required: ['action'],
    };
    this.shouldDefer = true;
  }

  isReadOnly(input) { return input?.action === 'screenshot'; }
  isConcurrencySafe() { return false; }
  isDestructive(input) {
    return ['click', 'double_click', 'right_click', 'type', 'key'].includes(input?.action);
  }

  async checkPermissions(input, context) {
    if (context.permissionMode === 'readonly') {
      if (input.action !== 'screenshot') {
        return { behavior: 'deny', message: 'Computer use writes require non-readonly mode.' };
      }
    }
    if (input.action !== 'screenshot' && context.permissionMode !== 'bypassPermissions') {
      return {
        behavior: 'ask',
        message: `Allow computer_use: ${input.action}${input.x !== undefined ? ` at (${input.x},${input.y})` : ''}${input.text ? `: "${input.text}"` : ''}?`,
      };
    }
    return { behavior: 'allow' };
  }

  async call(input, context) {
    const release = await acquireComputerUseLock(context.agentId || 'main');
    try {
      switch (input.action) {
        case 'screenshot': {
          const base64 = await screenshot();
          return {
            output: `Screenshot taken. Image data (base64 PNG, ${base64.length} chars):`,
            image: base64,
          };
        }
        case 'move_mouse':
          await moveMouse(input.x, input.y);
          return { output: `Mouse moved to (${input.x}, ${input.y}).` };
        case 'click':
          await click(input.x, input.y, 1);
          return { output: `Left-clicked at (${input.x}, ${input.y}).` };
        case 'double_click':
          await click(input.x, input.y, 1);
          await new Promise(r => setTimeout(r, 100));
          await click(input.x, input.y, 1);
          return { output: `Double-clicked at (${input.x}, ${input.y}).` };
        case 'right_click':
          await click(input.x, input.y, 3);
          return { output: `Right-clicked at (${input.x}, ${input.y}).` };
        case 'type':
          await typeText(input.text || '');
          return { output: `Typed: "${input.text}".` };
        case 'key':
          await pressKey(input.text || '');
          return { output: `Pressed key: ${input.text}.` };
        case 'scroll':
          await scroll(input.x, input.y, input.direction, input.amount);
          return { output: `Scrolled ${input.direction} ${input.amount}x at (${input.x}, ${input.y}).` };
        default:
          return { output: `Unknown action: ${input.action}` };
      }
    } finally {
      release();
    }
  }
}

module.exports = { ComputerUseTool };
```

Add `EUCLID_COMPUTER_USE=false` to `.env.example`. Set `shouldDefer: true` so it only appears after `tool_search('computer')`. On Linux, install `xdotool` and `scrot`: `apt-get install -y xdotool scrot`. The tool requires a display — set `DISPLAY=:0` in the environment for headless Linux.

---

## Appendix U — AutoDream: Background Memory Consolidation

**Source**: `src/services/autoDream/autoDream.ts`, `src/services/autoDream/config.ts`, `src/services/autoDream/consolidationLock.ts`, `src/services/autoDream/consolidationPrompt.ts`, `src/tasks/DreamTask/DreamTask.ts`

AutoDream is one of the most innovative features in Claude Code. It runs a **background consolidation pass** over all accumulated memories at a configurable interval (default: every 24 hours). During consolidation, a fast model reads all stored memories, removes duplicates, resolves contradictions, and produces a clean, condensed memory store. This prevents memory drift (where older incorrect facts stay in the store alongside corrected ones) and keeps the memory injection in the system prompt concise.

The `DreamTask` is the background task type that runs AutoDream — it is the `dream` task type from `Task.ts`'s enum. It runs in the background, writing progress to its output file.

```javascript
// src/services/autoDream/consolidation.js
'use strict';

const { complete } = require('../ollama/client');
const { getModel } = require('../../utils/modelRouter');
const { db } = require('../../db');
const fs = require('fs');
const path = require('path');
const os = require('os');

// Config from autoDream/config.ts
const AUTODREAM_CONFIG = {
  enabled: process.env.EUCLID_AUTODREAM !== 'false',
  intervalMs: parseInt(process.env.EUCLID_AUTODREAM_INTERVAL_MS || String(24 * 60 * 60 * 1000)), // 24h
  maxMemoriesPerPass: 500,
  consolidationModel: process.env.EUCLID_AUTODREAM_MODEL || '', // falls back to getModel('compact')
};

// Lock: only one consolidation pass at a time
let consolidationRunning = false;

const CONSOLIDATION_PROMPT = `You are a memory consolidation assistant. Your job is to clean up a set of stored memory entries.

For each group of memory entries about the same project or topic:
1. Merge duplicate or redundant facts into one clean statement
2. Resolve contradictions by keeping the most recent/accurate information
3. Remove information that is no longer relevant (e.g. "tried X, failed" when X was later fixed)
4. Preserve unique decisions, architecture choices, and key facts
5. Keep the total count below 80% of the input count

Output ONLY valid JSON:
{
  "consolidated": [
    { "namespace": "string", "content": "string", "importance": "high|medium|low" },
    ...
  ],
  "removed_count": 0,
  "merged_count": 0,
  "reasoning": "Brief explanation of what was consolidated"
}`;

/**
 * Run a full AutoDream consolidation pass.
 * This is called by the background cron scheduler.
 */
async function runConsolidationPass(sessionId) {
  if (consolidationRunning) {
    return { skipped: true, reason: 'Consolidation already running' };
  }

  consolidationRunning = true;
  const startTime = Date.now();

  try {
    // Load memories for this session
    const memories = db.prepare(`
      SELECT id, namespace, content, created_at
      FROM memory
      WHERE session_id = ? OR session_id = ''
      ORDER BY created_at DESC
      LIMIT ?
    `).all(sessionId || '', AUTODREAM_CONFIG.maxMemoriesPerPass);

    if (memories.length < 10) {
      return { skipped: true, reason: 'Too few memories to consolidate' };
    }

    // Group by namespace for the prompt
    const grouped = {};
    for (const mem of memories) {
      const ns = mem.namespace || 'general';
      if (!grouped[ns]) grouped[ns] = [];
      grouped[ns].push(mem.content);
    }

    const memoryText = Object.entries(grouped)
      .map(([ns, items]) => `## Namespace: ${ns}\n${items.join('\n')}`)
      .join('\n\n');

    const result = await complete({
      model: AUTODREAM_CONFIG.consolidationModel || getModel('compact'),
      messages: [
        { role: 'system', content: CONSOLIDATION_PROMPT },
        { role: 'user', content: `Consolidate these ${memories.length} memory entries:\n\n${memoryText}` },
      ],
      options: { temperature: 0.1, num_predict: 4096 },
    });

    const content = result?.message?.content || '';
    const json = content.replace(/```json\s*/g, '').replace(/```/g, '').trim();
    const parsed = JSON.parse(json);

    // Delete old memories for this session and write consolidated ones
    db.prepare('DELETE FROM memory WHERE session_id = ?').run(sessionId || '');

    const insertStmt = db.prepare(
      'INSERT INTO memory (id, namespace, content, session_id, created_at) VALUES (?, ?, ?, ?, ?)'
    );

    const { v4: uuidv4 } = require('uuid');
    for (const mem of parsed.consolidated) {
      insertStmt.run(uuidv4(), mem.namespace, mem.content, sessionId || '', Date.now());
    }

    const durationMs = Date.now() - startTime;

    // Log the consolidation run
    db.prepare(`
      INSERT INTO autodream_runs (session_id, input_count, output_count, removed_count, merged_count, duration_ms, reasoning, created_at)
      VALUES (?, ?, ?, ?, ?, ?, ?, ?)
    `).run(
      sessionId || '',
      memories.length,
      parsed.consolidated.length,
      parsed.removed_count || 0,
      parsed.merged_count || 0,
      durationMs,
      parsed.reasoning || '',
      Date.now()
    );

    return {
      inputCount: memories.length,
      outputCount: parsed.consolidated.length,
      removedCount: parsed.removed_count || 0,
      mergedCount: parsed.merged_count || 0,
      durationMs,
      reasoning: parsed.reasoning,
    };

  } catch (err) {
    console.error('[autodream] Consolidation failed:', err.message);
    return { error: err.message };
  } finally {
    consolidationRunning = false;
  }
}

/**
 * Schedule AutoDream to run every N hours. Called at server startup.
 */
function scheduleAutoDream() {
  if (!AUTODREAM_CONFIG.enabled) return;

  const runAndReschedule = async () => {
    console.log('[autodream] Running background consolidation pass...');
    await runConsolidationPass(null); // null = all sessions
    setTimeout(runAndReschedule, AUTODREAM_CONFIG.intervalMs);
  };

  // First run after 30 minutes (allow startup to settle)
  setTimeout(runAndReschedule, 30 * 60 * 1000);
}

module.exports = { runConsolidationPass, scheduleAutoDream, AUTODREAM_CONFIG };
```

Add the `autodream_runs` table to the schema:

```sql
CREATE TABLE IF NOT EXISTS autodream_runs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id TEXT DEFAULT '',
  input_count INTEGER DEFAULT 0,
  output_count INTEGER DEFAULT 0,
  removed_count INTEGER DEFAULT 0,
  merged_count INTEGER DEFAULT 0,
  duration_ms INTEGER DEFAULT 0,
  reasoning TEXT DEFAULT '',
  created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);
```

Call `scheduleAutoDream()` in `src/server.js` after the database initializes. Add `POST /api/autodream/trigger` for manual runs. Add `GET /api/autodream/history` to view past consolidation runs.

---

## Appendix V — Streaming Tool Executor

**Source**: `src/services/tools/StreamingToolExecutor.ts`, `src/services/tools/toolOrchestration.ts`, `src/services/tools/toolHooks.ts`, `src/services/tools/toolExecution.ts`

The `StreamingToolExecutor` is the advanced tool runner used when the `streamingToolExecution` feature gate is enabled. Unlike the basic `runTools` function (which waits for all tools to complete before returning), the StreamingToolExecutor yields partial results as tools complete, allowing the model to see some tool results while others are still running. This dramatically reduces the latency of turns with many parallel tool calls.

The `toolOrchestration.ts` module adds **tool dependency tracking** — if tool B depends on the output of tool A, it won't start until A completes. This enables structured parallelism without race conditions.

```javascript
// src/services/tools/streamingToolExecutor.js
'use strict';

const EventEmitter = require('events');

/**
 * StreamingToolExecutor — executes tools and yields results as they complete.
 *
 * Unlike runTools() which returns a Promise<Array>, this is an async generator
 * that yields { type: 'tool_result', id, name, output, error } as each tool finishes.
 * Tools with isConcurrencySafe() = true run in parallel; others run sequentially.
 */
async function* streamingToolExecutor(toolCallBlocks, tools, context, signal, onProgress) {
  const { loadTools } = require('../../tools');

  // Separate into concurrent and sequential groups
  const concurrent = [];
  const sequential = [];

  for (const block of toolCallBlocks) {
    const tool = findTool(tools, block.function?.name || block.name);
    if (!tool) {
      yield {
        type: 'tool_result',
        id: block.id,
        name: block.function?.name || block.name,
        output: `Tool not found.`,
        error: true,
      };
      continue;
    }
    (tool.isConcurrencySafe(parseInput(block)) ? concurrent : sequential).push({ block, tool });
  }

  // Run sequential tools first (they may modify state concurrent tools depend on)
  for (const { block, tool } of sequential) {
    if (signal?.aborted) break;
    const result = await executeSingleTool(tool, block, context, signal, onProgress);
    yield { type: 'tool_result', ...result };
  }

  if (signal?.aborted || !concurrent.length) return;

  // Run concurrent tools in parallel, yield each result as it arrives
  const promises = concurrent.map(({ block, tool }) =>
    executeSingleTool(tool, block, context, signal, onProgress)
  );

  // Use a queue to yield results in completion order
  const results = await Promise.allSettled(promises);
  for (const r of results) {
    if (r.status === 'fulfilled') {
      yield { type: 'tool_result', ...r.value };
    } else {
      yield { type: 'tool_result', id: 'unknown', name: 'unknown', output: r.reason?.message, error: true };
    }
  }
}

async function executeSingleTool(tool, block, context, signal, onProgress) {
  const input = parseInput(block);
  const id = block.id || require('uuid').v4();

  const startTime = Date.now();

  try {
    const validation = await tool.validateInput(input, context);
    if (!validation.valid) {
      return { id, name: tool.name, output: `Validation failed: ${validation.message}`, error: true };
    }

    const permission = await tool.checkPermissions(input, context);
    if (permission.behavior === 'deny') {
      return { id, name: tool.name, output: `Permission denied: ${permission.message}`, error: true };
    }

    if (permission.behavior === 'ask' && context.isNonInteractive) {
      return { id, name: tool.name, output: `Permission required (non-interactive): ${permission.message}`, error: true };
    }

    if (signal?.aborted) {
      return { id, name: tool.name, output: 'Aborted.', error: true };
    }

    const result = await tool.call(input, context, onProgress);

    const durationMs = Date.now() - startTime;

    // Audit log every successful tool call
    if (context.sessionId) {
      const { db } = require('../../db');
      db.prepare(`
        INSERT INTO tool_calls (tool_use_id, session_id, tool_name, input, output, duration_ms, error, created_at)
        VALUES (?, ?, ?, ?, ?, ?, 0, ?)
      `).run(id, context.sessionId, tool.name, JSON.stringify(input), String(result.output).slice(0, 10000), durationMs, Date.now());
    }

    return { id, name: tool.name, output: result.output, newMessages: result.newMessages, error: false };

  } catch (err) {
    return { id, name: tool.name, output: `Tool error: ${err.message}`, error: true };
  }
}

function findTool(tools, name) {
  return tools.find(t => t.name === name || (t.aliases || []).includes(name));
}

function parseInput(block) {
  const args = block.function?.arguments || block.input || '{}';
  try { return typeof args === 'string' ? JSON.parse(args) : args; } catch { return {}; }
}

module.exports = { streamingToolExecutor };
```

### Tool Dependency Graph

```javascript
// src/services/tools/toolOrchestration.js
'use strict';

/**
 * Analyze a set of tool calls and determine which can run in parallel
 * vs which must wait for others due to data dependencies.
 *
 * This is a simplified version of toolOrchestration.ts's dependency graph.
 * Dependency detection heuristic: if tool B references a file path that
 * tool A writes to, B depends on A.
 */
function buildDependencyGraph(toolCallBlocks, tools) {
  const writePaths = new Map(); // toolId → paths it writes
  const readPaths = new Map();  // toolId → paths it reads
  const deps = new Map();       // toolId → Set<toolId> (must complete before this)

  for (const block of toolCallBlocks) {
    const name = block.function?.name || block.name;
    const args = parseInput(block);

    // Heuristic: detect file paths in input
    const writes = new Set();
    const reads = new Set();

    if (name === 'file_write' || name === 'file_edit') {
      if (args.path) writes.add(args.path);
    }
    if (name === 'file_read' || name === 'read_file') {
      if (args.path) reads.add(args.path);
    }
    if (name === 'bash') {
      // Try to extract paths from bash commands (best-effort)
      const cmd = args.command || '';
      const pathMatches = cmd.match(/(?:>|>>)\s*([^\s;]+)/g) || [];
      for (const m of pathMatches) writes.add(m.replace(/^[>]+\s*/, ''));
    }

    writePaths.set(block.id, writes);
    readPaths.set(block.id, reads);
    deps.set(block.id, new Set());
  }

  // Build dependency edges: B depends on A if A writes what B reads
  for (const [bId, bReads] of readPaths) {
    for (const [aId, aWrites] of writePaths) {
      if (aId === bId) continue;
      for (const path of bReads) {
        if (aWrites.has(path)) {
          deps.get(bId).add(aId);
        }
      }
    }
  }

  return deps;
}

function parseInput(block) {
  const args = block.function?.arguments || block.input || '{}';
  try { return typeof args === 'string' ? JSON.parse(args) : args; } catch { return {}; }
}

module.exports = { buildDependencyGraph };
```

Wire the streaming tool executor into `queryLoop.js` behind the `gates.streamingToolExecution` flag from `queryConfig`. When enabled, replace the `runTools()` call with `for await (const result of streamingToolExecutor(...))` and collect into the `toolResults` array before continuing the loop.

---

## Appendix W — Full Plugin Architecture

**Source**: `src/utils/plugins/` (40 files), `src/services/plugins/`, `src/plugins/builtinPlugins.ts`

The plugin system is how Claude Code ships extensibility to developers. A plugin is a directory with a `package.json`, optional `SKILL.md`, optional `commands/`, optional `tools/`, optional `hooks/`, and optional `agents/`. Euclid implements the same architecture.

### Plugin Directory Format

```
~/.euclid/plugins/
  my-plugin/
    package.json          ← { name, version, description, author }
    SKILL.md              ← Optional: skill injected into system prompt
    commands/
      my-command.js       ← Exports { name, description, run(args, context) }
    tools/
      my-tool.js          ← Exports a BaseTool subclass
    hooks/
      pre-tool.js         ← Exports { event: 'PreToolUse', matcher, handler(input) }
    agents/
      my-agent.js         ← Exports { name, systemPrompt, model, temperature }
```

```javascript
// src/services/plugins/pluginLoader.js
'use strict';

const fs = require('fs');
const path = require('path');
const os = require('os');

const PLUGIN_DIRS = [
  path.join(os.homedir(), '.euclid', 'plugins'),
  path.join(process.cwd(), '.euclid', 'plugins'),
].filter(d => fs.existsSync(d));

const loadedPlugins = new Map();

/**
 * Scan all plugin directories and load each plugin.
 */
async function loadAllPlugins() {
  const results = { loaded: [], failed: [] };

  for (const pluginDir of PLUGIN_DIRS) {
    const entries = fs.readdirSync(pluginDir, { withFileTypes: true });

    for (const entry of entries) {
      if (!entry.isDirectory()) continue;
      const pluginPath = path.join(pluginDir, entry.name);

      try {
        const plugin = await loadPlugin(pluginPath);
        loadedPlugins.set(plugin.name, plugin);
        results.loaded.push(plugin.name);
      } catch (err) {
        results.failed.push({ name: entry.name, error: err.message });
      }
    }
  }

  return results;
}

async function loadPlugin(pluginPath) {
  const pkgPath = path.join(pluginPath, 'package.json');
  if (!fs.existsSync(pkgPath)) {
    throw new Error(`Missing package.json in ${pluginPath}`);
  }

  const pkg = JSON.parse(fs.readFileSync(pkgPath, 'utf8'));
  const plugin = {
    name: pkg.name,
    version: pkg.version || '0.0.0',
    description: pkg.description || '',
    path: pluginPath,
    commands: [],
    tools: [],
    hooks: [],
    agents: [],
    skill: null,
  };

  // Load skill
  const skillPath = path.join(pluginPath, 'SKILL.md');
  if (fs.existsSync(skillPath)) {
    plugin.skill = {
      name: pkg.name,
      description: pkg.description || '',
      content: fs.readFileSync(skillPath, 'utf8'),
    };
  }

  // Load commands
  const commandsDir = path.join(pluginPath, 'commands');
  if (fs.existsSync(commandsDir)) {
    for (const file of fs.readdirSync(commandsDir).filter(f => f.endsWith('.js'))) {
      try {
        const cmd = require(path.join(commandsDir, file));
        if (cmd.name && cmd.run) plugin.commands.push(cmd);
      } catch (err) {
        console.warn(`[plugin:${pkg.name}] Failed to load command ${file}: ${err.message}`);
      }
    }
  }

  // Load tools
  const toolsDir = path.join(pluginPath, 'tools');
  if (fs.existsSync(toolsDir)) {
    for (const file of fs.readdirSync(toolsDir).filter(f => f.endsWith('.js'))) {
      try {
        const ToolClass = require(path.join(toolsDir, file));
        // Support both class exports and factory function exports
        const tool = typeof ToolClass === 'function' && ToolClass.prototype.call
          ? new ToolClass()
          : typeof ToolClass === 'function'
            ? ToolClass()
            : ToolClass;
        if (tool && tool.name) plugin.tools.push(tool);
      } catch (err) {
        console.warn(`[plugin:${pkg.name}] Failed to load tool ${file}: ${err.message}`);
      }
    }
  }

  // Load hooks
  const hooksDir = path.join(pluginPath, 'hooks');
  if (fs.existsSync(hooksDir)) {
    for (const file of fs.readdirSync(hooksDir).filter(f => f.endsWith('.js'))) {
      try {
        const hook = require(path.join(hooksDir, file));
        if (hook.event && hook.handler) plugin.hooks.push(hook);
      } catch (err) {
        console.warn(`[plugin:${pkg.name}] Failed to load hook ${file}: ${err.message}`);
      }
    }
  }

  // Load agents
  const agentsDir = path.join(pluginPath, 'agents');
  if (fs.existsSync(agentsDir)) {
    for (const file of fs.readdirSync(agentsDir).filter(f => f.endsWith('.js'))) {
      try {
        const agent = require(path.join(agentsDir, file));
        if (agent.name && agent.systemPrompt) plugin.agents.push(agent);
      } catch (err) {
        console.warn(`[plugin:${pkg.name}] Failed to load agent ${file}: ${err.message}`);
      }
    }
  }

  return plugin;
}

function getLoadedPlugins() { return [...loadedPlugins.values()]; }
function getPluginTools() { return getLoadedPlugins().flatMap(p => p.tools); }
function getPluginCommands() { return getLoadedPlugins().flatMap(p => p.commands); }
function getPluginSkills() { return getLoadedPlugins().map(p => p.skill).filter(Boolean); }
function getPluginHooks() { return getLoadedPlugins().flatMap(p => p.hooks); }
function getPluginAgents() { return getLoadedPlugins().flatMap(p => p.agents); }

module.exports = {
  loadAllPlugins,
  getLoadedPlugins,
  getPluginTools,
  getPluginCommands,
  getPluginSkills,
  getPluginHooks,
  getPluginAgents,
};
```

### Plugin Management API

```javascript
// Add to src/routes/system.js
router.get('/api/plugins', auth, (req, res) => {
  const plugins = getLoadedPlugins().map(p => ({
    name: p.name,
    version: p.version,
    description: p.description,
    tools: p.tools.map(t => t.name),
    commands: p.commands.map(c => c.name),
    hasSkill: !!p.skill,
    agentCount: p.agents.length,
    hookCount: p.hooks.length,
  }));
  res.json({ plugins, count: plugins.length });
});

router.post('/api/plugins/reload', auth, async (req, res) => {
  const result = await loadAllPlugins();
  res.json({ ...result, total: result.loaded.length });
});

router.post('/api/plugins/install', auth, async (req, res) => {
  // Install a plugin from a git URL or npm package name
  const { source } = req.body;
  if (!source) return res.status(400).json({ error: 'source required (git URL or npm package)' });

  const pluginDir = path.join(os.homedir(), '.euclid', 'plugins');
  fs.mkdirSync(pluginDir, { recursive: true });

  try {
    if (source.startsWith('http') || source.startsWith('git@')) {
      const repoName = source.split('/').pop().replace('.git', '');
      execSync(`git clone "${source}" "${path.join(pluginDir, repoName)}"`, { stdio: 'inherit' });
    } else {
      // npm package — install and symlink
      execSync(`npm install --prefix "${pluginDir}" "${source}"`, { stdio: 'inherit' });
    }
    await loadAllPlugins();
    res.json({ success: true, source });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});
```

Wire plugin tools and skills into the main tool registry and skill selector. In `src/tools/index.js`, at the bottom of `loadTools()`, append `getPluginTools()`. In `src/services/skill/selector.js`, in `selectSkills()`, include `getPluginSkills()` in the candidate skill set. Call `loadAllPlugins()` in `src/server.js` during startup.

---

## Appendix X — Bundled Skills (16 Built-in Skills from src/skills/bundled/)

**Source**: `src/skills/bundled/` — `batch.ts`, `claudeApi.ts`, `debug.ts`, `keybindings.ts`, `loop.ts`, `loremIpsum.ts`, `remember.ts`, `scheduleRemoteAgents.ts`, `simplify.ts`, `skillify.ts`, `stuck.ts`, `updateConfig.ts`, `verify.ts`, `verifyContent.ts`, `claudeInChrome.ts`, `bundledSkills.ts`

These 16 skills are **bundled into the application binary** in Claude Code — they are always available without needing external skill files. Euclid adapts these as JavaScript skill objects exported from `src/skills/bundled/index.js`.

```javascript
// src/skills/bundled/index.js
'use strict';

// These skills are always available regardless of the skills/ directory.
// Each skill has: name, description, content (the SKILL.md text), alwaysActive

const BUNDLED_SKILLS = [
  {
    name: 'remember',
    description: 'Store and retrieve facts in persistent memory across sessions.',
    alwaysActive: false,
    content: `# Remember Skill
When the user says "remember X" or "don't forget X", store it in memory using the memory tools.
When starting a session, surface relevant memories.
Format: [MEMORY]: <fact>
Namespace all memories as 'user_prefs' unless the user specifies otherwise.`,
  },
  {
    name: 'verify',
    description: 'Run verification after completing any code change.',
    alwaysActive: true,
    content: `# Verify Skill
After EVERY code change, you MUST verify:
1. Run the code or its tests to confirm no regressions
2. Check that the change compiles/parses without errors
3. Run linting if a linter is configured
4. Report what you verified and what passed/failed
NEVER report "done" without actually running verification.`,
  },
  {
    name: 'debug',
    description: 'Systematic debugging approach for errors and unexpected behavior.',
    alwaysActive: false,
    content: `# Debug Skill
When debugging:
1. REPRODUCE the error first — don't guess
2. ADD logging/print statements to isolate the problem
3. FORM a hypothesis about the root cause
4. TEST the hypothesis with a minimal change
5. VERIFY the fix resolves the original error without regressions
Never add multiple changes at once when debugging — isolate variables.`,
  },
  {
    name: 'stuck',
    description: 'Recovery procedure when the agent is stuck in a loop or repeatedly failing.',
    alwaysActive: false,
    content: `# Stuck Recovery Skill
If you have tried the same approach 3+ times without success:
1. STOP and acknowledge you are stuck
2. DESCRIBE what you've tried and why it failed
3. CONSIDER a completely different approach
4. ASK the user for input if you cannot find a new approach
5. Do NOT retry the same failed approach again
Pattern recognition: same error + same fix = stuck. Break the loop.`,
  },
  {
    name: 'simplify',
    description: 'Preference for simple, minimal solutions over complex ones.',
    alwaysActive: true,
    content: `# Simplify Skill
Always prefer the simplest solution that solves the problem.
- Fewer lines of code is better
- Use standard library functions over custom implementations
- Avoid premature optimization
- One function, one responsibility
- Reject complexity that serves no current purpose (YAGNI)
If a simple approach exists, use it. Complexity must justify itself.`,
  },
  {
    name: 'loop',
    description: 'Keep working on a task in a loop until explicitly told to stop.',
    alwaysActive: false,
    content: `# Loop Skill
When asked to "keep going", "work in a loop", or "don't stop until done":
1. Continue working on the task without asking for confirmation between steps
2. Make progress on each iteration
3. Report what you did at the end of each iteration
4. Stop ONLY when: the task is complete, you hit an unresolvable blocker, or the user says stop
5. After each iteration, assess: "Is the task done?" If yes, stop and report completion.`,
  },
  {
    name: 'batch',
    description: 'Batch multiple operations together to minimize round trips.',
    alwaysActive: false,
    content: `# Batch Skill
When you have multiple operations of the same type:
- Read multiple files in a single parallel file_read batch
- Write multiple files with minimal iterations
- Run multiple searches simultaneously using parallel grep/glob
- Combine related bash commands with && or ; rather than separate calls
Never make sequential calls that could be parallel. Minimize total tool call count.`,
  },
  {
    name: 'skillify',
    description: 'Create a new SKILL.md from what you just learned in this session.',
    alwaysActive: false,
    content: `# Skillify Skill
When asked to "skillify" or "save this as a skill":
1. Summarize the key process, pattern, or knowledge from this session
2. Write it as a structured SKILL.md with: frontmatter, When to Activate section, Core Patterns section
3. Save it to skills/<skill-name>/SKILL.md
4. Register it with the skill loader
The goal: what would help future sessions on this same type of task?`,
  },
  {
    name: 'update-config',
    description: 'Update Euclid configuration and restart affected services.',
    alwaysActive: false,
    content: `# Update Config Skill
When updating configuration:
1. Read the current config first to understand existing settings
2. Make targeted changes — don't rewrite the entire config
3. Validate the config format before writing
4. Restart only the affected service, not the entire server
5. Verify the new config is loaded and working after restart`,
  },
  {
    name: 'verify-content',
    description: 'Verify that generated content meets quality and accuracy standards.',
    alwaysActive: false,
    content: `# Verify Content Skill
After generating content (docs, articles, code comments, READMEs):
1. Check factual accuracy against the actual codebase
2. Verify all code examples compile and run
3. Check for broken links or references
4. Ensure consistent terminology with the rest of the project
5. Read from the user's perspective — is this understandable?`,
  },
  {
    name: 'schedule-remote-agents',
    description: 'Schedule background work on remote agent infrastructure.',
    alwaysActive: false,
    content: `# Schedule Remote Agents Skill
When scheduling remote or background work:
1. Break the task into atomic, independently completable units
2. Assign each unit to a task or cron job with a clear success criterion
3. Set up output polling so you know when work is done
4. Handle failures: what happens if a remote agent fails partway through?
5. Aggregate results from all agents when all have completed`,
  },
  {
    name: 'claude-api',
    description: 'Patterns for integrating with the Claude/Anthropic API.',
    alwaysActive: false,
    content: `# Claude API Integration Skill
When integrating with Anthropic API:
- Use streaming for responses > 500 tokens
- Use tool_use for structured outputs
- Handle rate limits with exponential backoff
- Use prompt caching for long system prompts (cache_control: ephemeral)
- Track token usage for cost monitoring
- Use claude-sonnet-4-6 for everyday tasks, claude-opus-4-6 for complex reasoning
For Euclid itself: use the Ollama client, not the Anthropic API.`,
  },
];

// Filter to always-active skills for baseline system prompt injection
function getAlwaysActiveSkills() {
  return BUNDLED_SKILLS.filter(s => s.alwaysActive);
}

function getBundledSkill(name) {
  return BUNDLED_SKILLS.find(s => s.name === name) || null;
}

function getAllBundledSkills() {
  return BUNDLED_SKILLS;
}

module.exports = { BUNDLED_SKILLS, getAlwaysActiveSkills, getBundledSkill, getAllBundledSkills };
```

Wire bundled skills into the skill selector: in `src/services/skill/selector.js`, always include `getAlwaysActiveSkills()` in the active skill set. When selecting domain skills, also search bundled skills alongside file-based skills.

---

## Appendix Y — Permission System Depth

**Source**: `src/utils/permissions/` (24 files) — `yoloClassifier.ts`, `bashClassifier.ts`, `autoModeState.ts`, `dangerousPatterns.ts`, `shadowedRuleDetection.ts`, `denialTracking.ts`, `permissionExplainer.ts`, `permissionRuleParser.ts`, `shellRuleMatching.ts`

### YOLO Classifier (Bypass Mode)

The `yoloClassifier.ts` implements the **bypass permissions** mode — when enabled, the model's tool calls are automatically approved without human confirmation. This is the most dangerous mode and requires explicit opt-in. Euclid gates it behind a dedicated `EUCLID_ALLOW_BYPASS=true` environment variable AND a one-time confirmation dialog.

```javascript
// src/utils/permissions/yoloClassifier.js
'use strict';

const BYPASS_ENABLED = process.env.EUCLID_ALLOW_BYPASS === 'true';

// Regardless of bypass mode, some operations ALWAYS require explicit approval
const ALWAYS_REQUIRE_APPROVAL = new Set([
  'rm -rf',
  'dd if=',
  'mkfs',
  'format',
  ':(){:|:&};:',  // fork bomb
  'chmod 777 /',
  'sudo rm',
  'DROP TABLE',
  'DELETE FROM',
  'TRUNCATE',
]);

function shouldBypassPermission(toolName, input, permissionMode) {
  if (permissionMode !== 'bypassPermissions') return false;
  if (!BYPASS_ENABLED) return false;

  // Check for absolute no-bypass patterns
  const inputStr = JSON.stringify(input).toLowerCase();
  for (const pattern of ALWAYS_REQUIRE_APPROVAL) {
    if (inputStr.includes(pattern.toLowerCase())) {
      return false; // Always ask for these
    }
  }

  return true;
}

module.exports = { shouldBypassPermission, BYPASS_ENABLED };
```

### Bash Classifier

The `bashClassifier.ts` assigns a risk category to each bash command. This is more sophisticated than the simple `safetyScore()` function in Section 7.3 — it uses tree-sitter to parse the shell AST and classifies based on command semantics.

```javascript
// src/utils/permissions/bashClassifier.js
'use strict';

// Risk categories (matching Claude Code's classification)
const RISK = {
  SAFE: 'safe',           // Read-only: ls, cat, grep, find, git log
  LOW: 'low',             // Minor writes: touch, mkdir, git add
  MEDIUM: 'medium',       // Significant writes: npm install, git commit
  HIGH: 'high',           // Dangerous: rm, curl | sh, chmod, chown
  CRITICAL: 'critical',   // Irreversible: rm -rf, dd, mkfs, format
};

// Command classification rules — ordered from most to least specific
const CLASSIFICATIONS = [
  // Critical — irreversible system operations
  { pattern: /\brm\s+-[^-]*r[^-]*f|\brm\s+--force/i, risk: RISK.CRITICAL, reason: 'Recursive force delete' },
  { pattern: /\bdd\s+if=|mkfs\.|mformat\b|diskutil\s+eraseVolume/i, risk: RISK.CRITICAL, reason: 'Disk overwrite' },
  { pattern: /\b(chmod|chown)\s+777\s+\/|>\s*\/dev\/sd[a-z]/i, risk: RISK.CRITICAL, reason: 'System filesystem modification' },
  { pattern: /:\(\)\s*\{.*\|.*:.*&.*\}.*;.*:/i, risk: RISK.CRITICAL, reason: 'Fork bomb' },

  // High risk
  { pattern: /curl[^|]*\|\s*(bash|sh|python|node)/i, risk: RISK.HIGH, reason: 'Remote code execution via pipe' },
  { pattern: /wget[^-]*-[^-]*O[^-]*-[^|]*\|\s*(bash|sh)/i, risk: RISK.HIGH, reason: 'Remote code execution via wget' },
  { pattern: /\beval\s+.*base64/i, risk: RISK.HIGH, reason: 'Base64-decoded execution' },
  { pattern: /\bsudo\s+/i, risk: RISK.HIGH, reason: 'Elevated privileges' },
  { pattern: /\brm\s+/i, risk: RISK.HIGH, reason: 'File deletion' },

  // Medium risk
  { pattern: /\bnpm\s+(install|i|add|ci)\b/i, risk: RISK.MEDIUM, reason: 'Package installation' },
  { pattern: /\bgit\s+(commit|push|merge|rebase|reset\s+--hard)/i, risk: RISK.MEDIUM, reason: 'Git history modification' },
  { pattern: /\bpip\s+install\b/i, risk: RISK.MEDIUM, reason: 'Python package installation' },
  { pattern: />\s+[^\s>]/i, risk: RISK.MEDIUM, reason: 'File overwrite via redirect' },

  // Low risk
  { pattern: /\bgit\s+(add|status|diff|log|branch)\b/i, risk: RISK.LOW, reason: 'Git read/staging operation' },
  { pattern: /\b(mkdir|touch|cp|mv)\b/i, risk: RISK.LOW, reason: 'File system modification' },
  { pattern: /\bnpm\s+(run|test|build|lint)\b/i, risk: RISK.LOW, reason: 'Script execution' },

  // Safe — read operations
  { pattern: /^(ls|cat|grep|find|head|tail|wc|diff|git\s+log|git\s+show|echo|pwd|which|type|env|printenv)\b/i, risk: RISK.SAFE, reason: 'Read-only operation' },
];

function classifyBashCommand(command) {
  const trimmed = (command || '').trim();

  for (const rule of CLASSIFICATIONS) {
    if (rule.pattern.test(trimmed)) {
      return { risk: rule.risk, reason: rule.reason, command: trimmed };
    }
  }

  // Default: unknown commands are medium risk
  return { risk: RISK.MEDIUM, reason: 'Unknown command', command: trimmed };
}

function requiresApproval(command, permissionMode) {
  const { risk } = classifyBashCommand(command);

  if (permissionMode === 'bypassPermissions') {
    return risk === RISK.CRITICAL; // Only critical requires approval in bypass mode
  }

  if (permissionMode === 'readonly') {
    return risk !== RISK.SAFE; // Everything except safe requires approval in readonly
  }

  // Default: high and critical require approval
  return risk === RISK.HIGH || risk === RISK.CRITICAL;
}

module.exports = { classifyBashCommand, requiresApproval, RISK };
```

### Denial Tracking and Shadowed Rules

```javascript
// src/utils/permissions/denialTracking.js
'use strict';

const { db } = require('../../db');

// Track consecutive denials per tool — if a tool is denied 3+ times in a row,
// surface a warning to the user and suggest adding a permanent permission rule.
function trackDenial(sessionId, toolName, input, reason) {
  db.prepare(`
    INSERT INTO permission_decisions (session_id, tool_name, input_hash, decision, reason, created_at)
    VALUES (?, ?, ?, 'deny', ?, ?)
  `).run(sessionId, toolName, hashInput(input), reason, Date.now());
}

function trackApproval(sessionId, toolName, input, permanent) {
  db.prepare(`
    INSERT INTO permission_decisions (session_id, tool_name, input_hash, decision, permanent, created_at)
    VALUES (?, ?, ?, 'allow', ?, ?)
  `).run(sessionId, toolName, hashInput(input), permanent ? 1 : 0, Date.now());
}

function getRecentDenials(sessionId, toolName, windowMs = 5 * 60 * 1000) {
  const cutoff = Date.now() - windowMs;
  return db.prepare(`
    SELECT COUNT(*) as count FROM permission_decisions
    WHERE session_id = ? AND tool_name = ? AND decision = 'deny' AND created_at > ?
  `).get(sessionId, toolName, cutoff)?.count || 0;
}

/**
 * Detect shadowed rules: if a specific allow rule is effectively overridden by
 * a broader deny rule, warn the user. From shadowedRuleDetection.ts.
 */
function detectShadowedRules(rules) {
  const warnings = [];
  for (let i = 0; i < rules.length; i++) {
    for (let j = 0; j < rules.length; j++) {
      if (i === j) continue;
      const ruleA = rules[i]; // potential shadower
      const ruleB = rules[j]; // potentially shadowed

      if (ruleA.action === 'deny' && ruleB.action === 'allow') {
        // If A's pattern is broader than B's, B is shadowed
        if (ruleA.pattern === '*' || ruleB.pattern?.startsWith(ruleA.pattern || '')) {
          warnings.push({
            shadowed: ruleB,
            shadowedBy: ruleA,
            message: `Rule "${ruleB.pattern}" is shadowed by "${ruleA.pattern}" (deny takes precedence).`,
          });
        }
      }
    }
  }
  return warnings;
}

function hashInput(input) {
  const crypto = require('crypto');
  return crypto.createHash('md5').update(JSON.stringify(input)).digest('hex').slice(0, 16);
}

module.exports = { trackDenial, trackApproval, getRecentDenials, detectShadowedRules };
```

Add the `permission_decisions` table to the schema:

```sql
CREATE TABLE IF NOT EXISTS permission_decisions (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id TEXT NOT NULL,
  tool_name TEXT NOT NULL,
  input_hash TEXT NOT NULL,
  decision TEXT NOT NULL CHECK(decision IN ('allow','deny')),
  reason TEXT DEFAULT '',
  permanent INTEGER DEFAULT 0,
  created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);
CREATE INDEX IF NOT EXISTS idx_perm_decisions ON permission_decisions(session_id, tool_name, decision, created_at DESC);
```

---

## Appendix Z — Context Analysis and Compaction Variants

**Source**: `src/utils/analyzeContext.ts`, `src/utils/contextAnalysis.ts`, `src/services/compact/apiMicrocompact.ts`, `src/services/compact/microCompact.ts`, `src/services/compact/grouping.ts`, `src/services/compact/sessionMemoryCompact.ts`, `src/services/compact/compactWarningHook.ts`

### Microcompact (API-Level Prompt Cache Compaction)

`apiMicrocompact.ts` implements a technique where **repetitive tool result patterns** in the message history are compressed before being sent to the model. For example, if the model called `file_read` on the same file 5 times and got the same content, only one copy is kept in the history; the others are replaced with `[DUPLICATE_TOOL_RESULT: file_read(<path>), see turn N]`. This reduces prompt size without full conversation compaction.

```javascript
// src/services/compact/microCompact.js
'use strict';

const crypto = require('crypto');

const DEDUP_MAX_RESULT_LENGTH = 5000; // chars — only dedup results this long or longer
const DEDUP_KEEP_FIRST_N = 1;         // Keep the first N copies, replace the rest

/**
 * Microcompact: deduplicate identical tool results within the message history.
 * Returns a new messages array with duplicates replaced by references.
 *
 * This is distinct from autoCompact (which summarizes the conversation).
 * Microcompact only removes redundancy, not meaning.
 */
function microCompact(messages) {
  // Build a map: resultHash → { firstTurn, firstIndex }
  const seenResults = new Map();
  const result = [];
  let deduplicatedCount = 0;

  for (let i = 0; i < messages.length; i++) {
    const msg = messages[i];

    if (msg.role !== 'user' || !Array.isArray(msg.content)) {
      result.push(msg);
      continue;
    }

    // Process tool result blocks in user messages
    const newContent = msg.content.map(block => {
      if (block.type !== 'tool_result') return block;

      const resultText = typeof block.content === 'string'
        ? block.content
        : JSON.stringify(block.content);

      if (resultText.length < DEDUP_MAX_RESULT_LENGTH) return block;

      const hash = crypto.createHash('md5').update(resultText).digest('hex');
      const key = `${block.tool_name || 'tool'}:${hash}`;

      if (!seenResults.has(key)) {
        seenResults.set(key, { firstTurn: i, count: 1 });
        return block; // Keep the first occurrence
      }

      const seen = seenResults.get(key);
      seen.count++;
      deduplicatedCount++;

      // Replace with a reference to the original
      return {
        type: 'tool_result',
        tool_use_id: block.tool_use_id,
        content: `[COMPACTED: Identical result to turn ${seen.firstTurn}. ${resultText.length} chars deduplicated.]`,
      };
    });

    result.push({ ...msg, content: newContent });
  }

  return { messages: result, deduplicatedCount };
}

/**
 * Grouping compaction: replace a sequence of consecutive read-only tool calls
 * (file_read, grep, glob, ls) with a single grouped summary.
 * Source: src/services/compact/grouping.ts
 */
function groupCompact(messages) {
  // This is a simplified version — full implementation uses AST grouping
  const READ_ONLY_TOOLS = new Set(['file_read', 'grep', 'glob', 'ls', 'web_fetch']);
  const result = [];
  let groupBuffer = [];
  let groupedCount = 0;

  function flushGroup() {
    if (groupBuffer.length < 3) {
      result.push(...groupBuffer);
    } else {
      // Summarize the group
      const summary = groupBuffer.map(m => {
        const toolName = m.content?.find?.(b => b.type === 'tool_use')?.name || 'tool';
        return `[${toolName}]`;
      }).join(' → ');

      result.push({
        ...groupBuffer[0],
        _grouped: true,
        _summary: `[GROUPED: ${groupBuffer.length} read-only tool calls: ${summary}]`,
      });
      groupedCount += groupBuffer.length - 1;
    }
    groupBuffer = [];
  }

  for (const msg of messages) {
    const isReadOnlyToolMsg = msg.role === 'assistant' &&
      Array.isArray(msg.content) &&
      msg.content.every(b => b.type !== 'tool_use' || READ_ONLY_TOOLS.has(b.name));

    if (isReadOnlyToolMsg) {
      groupBuffer.push(msg);
    } else {
      flushGroup();
      result.push(msg);
    }
  }

  flushGroup();
  return { messages: result, groupedCount };
}

/**
 * Session memory compaction: after a full autoCompact, update the session memory
 * summary based on what was compacted.
 * Source: src/services/compact/sessionMemoryCompact.ts
 */
async function sessionMemoryCompact(compactedMessages, sessionId, model) {
  const { complete } = require('../ollama/client');
  const { getModel } = require('../../utils/modelRouter');

  if (!compactedMessages.length) return '';

  const transcript = compactedMessages
    .filter(m => m.role === 'user' || m.role === 'assistant')
    .map(m => `[${m.role}]: ${typeof m.content === 'string' ? m.content : JSON.stringify(m.content)}`)
    .join('\n')
    .slice(0, 8000);

  const result = await complete({
    model: model || getModel('compact'),
    messages: [{
      role: 'user',
      content: `Summarize the key facts, decisions, and progress from this conversation that should be remembered:

${transcript}

Output a brief bulleted list (max 10 bullets, each < 100 chars).`,
    }],
    options: { temperature: 0.1, num_predict: 512 },
  });

  return result?.message?.content || '';
}

module.exports = { microCompact, groupCompact, sessionMemoryCompact };
```

Wire `microCompact` into `queryLoop.js`: call it in Phase 1 after `applyToolResultBudget` but before the model call. The output feeds into the context fill calculation. If `deduplicatedCount > 5`, emit a `{ type: 'system', message: 'Microcompacted N duplicate tool results' }` event.

---

## Appendix AA — BriefTool, ConfigTool, and WebSearchTool

**Source**: `src/tools/BriefTool/`, `src/tools/ConfigTool/`, `src/tools/WebSearchTool/`

### BriefTool (File Attachment and Upload)

`BriefTool` in Claude Code allows the model to create "briefs" — structured summaries of uploaded files or documents that are attached to the conversation. The tool extracts text from PDFs, images (via OCR), and office documents, then compresses the content into a brief that fits in the context window.

```javascript
// src/tools/brief/briefTool.js
'use strict';

const { complete } = require('../../services/ollama/client');
const { getModel } = require('../../utils/modelRouter');
const multer = require('multer');
const fs = require('fs');
const path = require('path');
const os = require('os');

const BRIEF_MAX_CONTENT_CHARS = 20_000;

class BriefTool extends BaseTool {
  constructor() {
    super();
    this.name = 'create_brief';
    this.description = 'Create a structured brief from an uploaded file (PDF, text, code, image). ' +
      'The brief summarizes the file contents in a context-efficient format for the conversation.';
    this.inputSchema = {
      type: 'object',
      properties: {
        file_path: { type: 'string', description: 'Path to a file already on disk to brief.' },
        content: { type: 'string', description: 'Raw text content to brief directly.' },
        focus: { type: 'string', description: 'Optional: what aspect to focus the brief on.' },
        max_length: { type: 'integer', description: 'Max output length in chars (default 3000).' },
      },
    };
  }

  isReadOnly() { return true; }
  isConcurrencySafe() { return true; }

  async call(input, context) {
    let rawContent = input.content || '';

    if (input.file_path && !rawContent) {
      const filePath = path.resolve(context.cwd, input.file_path);
      if (!fs.existsSync(filePath)) {
        return { output: `File not found: ${input.file_path}` };
      }

      const ext = path.extname(filePath).toLowerCase();
      const stat = fs.statSync(filePath);

      if (stat.size > 50 * 1024 * 1024) {
        return { output: `File too large for briefing (${(stat.size / 1024 / 1024).toFixed(1)}MB). Max 50MB.` };
      }

      if (['.txt', '.md', '.js', '.ts', '.py', '.go', '.rs', '.java', '.c', '.cpp', '.h', '.json', '.yaml', '.yml', '.toml', '.sh'].includes(ext)) {
        rawContent = fs.readFileSync(filePath, 'utf8');
      } else if (ext === '.pdf') {
        rawContent = await extractPdfText(filePath);
      } else {
        return { output: `Cannot brief file type ${ext}. Supported: text, code, PDF.` };
      }
    }

    if (!rawContent) return { output: 'No content to brief.' };

    // Truncate if needed
    if (rawContent.length > BRIEF_MAX_CONTENT_CHARS) {
      rawContent = rawContent.slice(0, BRIEF_MAX_CONTENT_CHARS) + '\n[TRUNCATED]';
    }

    const maxLen = input.max_length || 3000;
    const focusStr = input.focus ? `Focus on: ${input.focus}` : 'Provide a general summary.';

    const result = await complete({
      model: getModel('compact'),
      messages: [{
        role: 'user',
        content: `Create a structured brief of this content. ${focusStr}
The brief should include: purpose/overview, key facts/findings, main components, and any important caveats.
Keep it under ${maxLen} characters.

CONTENT:
${rawContent}`,
      }],
      options: { temperature: 0.1, num_predict: 1024 },
    });

    const brief = result?.message?.content || '(No brief generated)';

    return {
      output: `Brief created (${brief.length} chars):\n\n${brief}`,
    };
  }
}

async function extractPdfText(filePath) {
  try {
    const { execSync } = require('child_process');
    return execSync(`pdftotext "${filePath}" -`, { encoding: 'utf8', maxBuffer: 10 * 1024 * 1024 });
  } catch {
    return `[PDF text extraction failed. Install poppler-utils: apt-get install poppler-utils]`;
  }
}

module.exports = { BriefTool };
```

### ConfigTool (Runtime Settings Management)

`ConfigTool` lets the model read and write Euclid's own runtime configuration — changing the model, adjusting thresholds, enabling/disabling features — without restarting the server.

```javascript
// src/tools/config/configTool.js
'use strict';

const { db } = require('../../db');

// Settings the model is allowed to change at runtime
const CONFIGURABLE_SETTINGS = {
  'model.main': { type: 'string', description: 'Primary model name', requiresRestart: false },
  'model.compact': { type: 'string', description: 'Compaction model name', requiresRestart: false },
  'context.compactThreshold': { type: 'number', description: 'Context fill % to trigger compaction (0.6-0.95)', min: 0.6, max: 0.95 },
  'memory.extractEvery': { type: 'integer', description: 'Extract memories every N turns', min: 1, max: 20 },
  'permissions.mode': { type: 'string', enum: ['default', 'readonly', 'bypassPermissions'], description: 'Permission mode' },
  'debug.verbose': { type: 'boolean', description: 'Enable verbose debug logging' },
};

class ConfigTool extends BaseTool {
  constructor() {
    super();
    this.name = 'euclid_config';
    this.description = 'Read or update Euclid runtime configuration. ' +
      'Use to change models, thresholds, and feature flags without restarting.';
    this.inputSchema = {
      type: 'object',
      properties: {
        action: { type: 'string', enum: ['get', 'set', 'list'] },
        key: { type: 'string', description: 'Setting key (for get/set).' },
        value: { description: 'New value (for set).' },
      },
      required: ['action'],
    };
  }

  isReadOnly(input) { return input?.action === 'get' || input?.action === 'list'; }
  isConcurrencySafe() { return false; }

  async call(input) {
    switch (input.action) {
      case 'list': {
        const lines = Object.entries(CONFIGURABLE_SETTINGS).map(([k, v]) => {
          const current = db.prepare('SELECT value FROM prefs WHERE key = ?').get(k);
          return `${k} = ${current?.value || '(default)'} [${v.type}] — ${v.description}`;
        });
        return { output: `Configurable settings:\n\n${lines.join('\n')}` };
      }

      case 'get': {
        if (!input.key) return { output: 'Key required for get.' };
        const row = db.prepare('SELECT value FROM prefs WHERE key = ?').get(input.key);
        const meta = CONFIGURABLE_SETTINGS[input.key];
        if (!meta) return { output: `Unknown setting: ${input.key}` };
        return { output: `${input.key} = ${row?.value || '(default)'}\nDescription: ${meta.description}` };
      }

      case 'set': {
        if (!input.key || input.value === undefined) {
          return { output: 'Key and value required for set.' };
        }
        const meta = CONFIGURABLE_SETTINGS[input.key];
        if (!meta) {
          return { output: `Unknown setting: ${input.key}. Use action: 'list' to see available settings.` };
        }

        // Type validation
        const val = String(input.value);
        if (meta.type === 'number' || meta.type === 'integer') {
          const n = Number(val);
          if (isNaN(n)) return { output: `${input.key} must be a number.` };
          if (meta.min !== undefined && n < meta.min) return { output: `${input.key} must be >= ${meta.min}.` };
          if (meta.max !== undefined && n > meta.max) return { output: `${input.key} must be <= ${meta.max}.` };
        }
        if (meta.enum && !meta.enum.includes(val)) {
          return { output: `${input.key} must be one of: ${meta.enum.join(', ')}` };
        }

        db.prepare('INSERT OR REPLACE INTO prefs (key, value, updated_at) VALUES (?, ?, ?)').run(input.key, val, Date.now());

        return { output: `Set ${input.key} = ${val}${meta.requiresRestart ? ' (requires server restart)' : ' (live)'}` };
      }

      default:
        return { output: `Unknown action: ${input.action}` };
    }
  }
}

module.exports = { ConfigTool };
```

Update the `prefs` table to support arbitrary key-value pairs (currently it's a single-row JSON blob in the schema — change to a proper key-value table):

```sql
-- Replace the existing prefs table with:
CREATE TABLE IF NOT EXISTS prefs (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL,
  updated_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);
```

### WebSearchTool (Live Web Search)

```javascript
// src/tools/webSearch/webSearchTool.js
'use strict';

// Supported backends: SearXNG (self-hosted), Brave Search API, DuckDuckGo (scrape)
const SEARCH_BACKEND = process.env.EUCLID_SEARCH_BACKEND || 'searxng';
const SEARXNG_URL = process.env.SEARXNG_URL || 'http://localhost:8080';
const BRAVE_API_KEY = process.env.BRAVE_SEARCH_API_KEY || '';

class WebSearchTool extends BaseTool {
  constructor() {
    super();
    this.name = 'web_search';
    this.description = 'Search the web for current information. Returns title, URL, and snippet for top results. ' +
      'Use for: recent events, documentation versions, package releases, error messages.';
    this.inputSchema = {
      type: 'object',
      properties: {
        query: { type: 'string', description: 'Search query (concise, 3-8 words).' },
        num_results: { type: 'integer', description: 'Number of results (default 5, max 10).', default: 5 },
        language: { type: 'string', description: 'Language code (default: en).', default: 'en' },
      },
      required: ['query'],
    };
  }

  isReadOnly() { return true; }
  isConcurrencySafe() { return true; }

  async call(input) {
    const numResults = Math.min(input.num_results || 5, 10);

    try {
      let results;

      if (SEARCH_BACKEND === 'brave' && BRAVE_API_KEY) {
        results = await searchBrave(input.query, numResults, input.language);
      } else if (SEARCH_BACKEND === 'searxng') {
        results = await searchSearXNG(input.query, numResults, input.language);
      } else {
        results = await searchDuckDuckGo(input.query, numResults);
      }

      if (!results.length) {
        return { output: `No results found for: "${input.query}"` };
      }

      const formatted = results.map((r, i) =>
        `[${i + 1}] ${r.title}\n    ${r.url}\n    ${r.snippet}`
      ).join('\n\n');

      return { output: `Search results for "${input.query}":\n\n${formatted}` };

    } catch (err) {
      return { output: `Search failed: ${err.message}. Backend: ${SEARCH_BACKEND}` };
    }
  }
}

async function searchBrave(query, num, lang) {
  const res = await fetch(
    `https://api.search.brave.com/res/v1/web/search?q=${encodeURIComponent(query)}&count=${num}&country=${lang}`,
    { headers: { 'Accept': 'application/json', 'X-Subscription-Token': BRAVE_API_KEY } }
  );
  const data = await res.json();
  return (data.web?.results || []).map(r => ({
    title: r.title,
    url: r.url,
    snippet: r.description || '',
  }));
}

async function searchSearXNG(query, num, lang) {
  const url = `${SEARXNG_URL}/search?q=${encodeURIComponent(query)}&format=json&language=${lang}&safesearch=0&categories=general`;
  const res = await fetch(url, { headers: { 'Accept': 'application/json' } });
  const data = await res.json();
  return (data.results || []).slice(0, num).map(r => ({
    title: r.title,
    url: r.url,
    snippet: r.content || '',
  }));
}

async function searchDuckDuckGo(query, num) {
  // DuckDuckGo Instant Answer API (limited but free, no key)
  const url = `https://api.duckduckgo.com/?q=${encodeURIComponent(query)}&format=json&no_html=1&skip_disambig=1`;
  const res = await fetch(url, { headers: { 'Accept': 'application/json' } });
  const data = await res.json();

  const results = [];
  if (data.AbstractText) {
    results.push({ title: data.Heading || query, url: data.AbstractURL, snippet: data.AbstractText });
  }
  for (const rel of (data.RelatedTopics || []).slice(0, num - 1)) {
    if (rel.FirstURL && rel.Text) {
      results.push({ title: rel.Text.slice(0, 60), url: rel.FirstURL, snippet: rel.Text });
    }
  }
  return results.slice(0, num);
}

module.exports = { WebSearchTool };
```

Add `EUCLID_SEARCH_BACKEND=searxng`, `SEARXNG_URL=http://localhost:8080`, `BRAVE_SEARCH_API_KEY=` to `.env.example`. To self-host SearXNG: `docker run -d -p 8080:8080 searxng/searxng`.

---

## Appendix AB — Session Backgrounding and Teleport

**Source**: `src/utils/background/remote/`, `src/utils/teleport.tsx`, `src/utils/teleport/`, `src/hooks/useSessionBackgrounding.ts`, `src/services/awaySummary.ts`

### Session Backgrounding

Session backgrounding allows a session to be detached from its current client and resumed later. When a session is backgrounded, it continues running its current agentic loop — tool calls keep executing even after the browser tab closes. The `awaySummary.ts` service generates a summary of what happened while the user was away, injected at the start of the next interaction.

```javascript
// src/services/session/backgrounding.js
'use strict';

const { db } = require('../../db');
const { complete } = require('../ollama/client');
const { getModel } = require('../../utils/modelRouter');

// Sessions that are backgrounded (still running but no active client)
const backgroundedSessions = new Set();

/**
 * Background a session — detach it from its SSE client but keep the loop running.
 * The loop continues writing to the DB; the client can reconnect later.
 */
function backgroundSession(sessionId) {
  backgroundedSessions.add(sessionId);
  db.prepare('UPDATE sessions SET backgrounded = 1, backgrounded_at = ? WHERE id = ?')
    .run(Date.now(), sessionId);
}

/**
 * Foreground a backgrounded session — generate an away summary for the user.
 */
async function forgroundSession(sessionId) {
  if (!backgroundedSessions.has(sessionId)) return null;

  backgroundedSessions.delete(sessionId);
  db.prepare('UPDATE sessions SET backgrounded = 0 WHERE id = ?').run(sessionId);

  return await generateAwaySummary(sessionId);
}

/**
 * Generate a summary of what happened while the user was away.
 * Source: src/services/awaySummary.ts
 */
async function generateAwaySummary(sessionId) {
  // Get messages generated since the session was backgrounded
  const session = db.prepare('SELECT backgrounded_at FROM sessions WHERE id = ?').get(sessionId);
  if (!session?.backgrounded_at) return null;

  const recentMessages = db.prepare(`
    SELECT role, content FROM messages
    WHERE session_id = ? AND created_at > ?
    ORDER BY created_at ASC
    LIMIT 50
  `).all(sessionId, session.backgrounded_at);

  if (!recentMessages.length) return null;

  const transcript = recentMessages
    .map(m => `[${m.role}]: ${(m.content || '').slice(0, 500)}`)
    .join('\n');

  const result = await complete({
    model: getModel('compact'),
    messages: [{
      role: 'user',
      content: `Summarize what happened while you were working in the background. 
Be brief (3-5 bullet points). Focus on: what was accomplished, any issues encountered, what needs user input next.

TRANSCRIPT:
${transcript}`,
    }],
    options: { temperature: 0.1, num_predict: 512 },
  });

  return result?.message?.content || null;
}

module.exports = { backgroundSession, forgroundSession, generateAwaySummary, backgroundedSessions };
```

Add `backgrounded INTEGER DEFAULT 0` and `backgrounded_at INTEGER` to the `sessions` table. Add `POST /api/sessions/:id/background` and `POST /api/sessions/:id/foreground` routes. When foregrounding, inject the away summary as a system message at the start of the resumed session.

### Teleport (Session Cloning to Remote)

Teleport allows a local session to be "teleported" to a remote cloud environment — the conversation history, CWD, and working files are bundled as a git bundle and pushed to a remote Euclid instance.

```javascript
// src/services/session/teleport.js
'use strict';

const { execSync } = require('child_process');
const path = require('path');
const os = require('os');
const fs = require('fs');
const { db } = require('../../db');

/**
 * Create a teleport bundle: a git bundle of the current working directory
 * plus the serialized session state.
 */
async function createTeleportBundle(sessionId, targetUrl) {
  const session = db.prepare('SELECT * FROM sessions WHERE id = ?').get(sessionId);
  if (!session) throw new Error(`Session ${sessionId} not found.`);

  const messages = db.prepare('SELECT * FROM messages WHERE session_id = ? ORDER BY created_at').all(sessionId);
  const memories = db.prepare("SELECT * FROM memory WHERE session_id = ?").all(sessionId);

  const bundleDir = path.join(os.tmpdir(), `euclid-teleport-${sessionId.slice(0, 8)}`);
  fs.mkdirSync(bundleDir, { recursive: true });

  // Create git bundle of working directory
  const cwd = session.cwd || process.cwd();
  const gitBundlePath = path.join(bundleDir, 'repo.bundle');

  try {
    execSync(`git bundle create "${gitBundlePath}" HEAD`, { cwd, stdio: 'ignore' });
  } catch {
    // Not a git repo — create a tar instead
    const tarPath = path.join(bundleDir, 'cwd.tar.gz');
    execSync(`tar czf "${tarPath}" -C "${cwd}" .`, { stdio: 'ignore' });
  }

  // Write session state
  const sessionState = {
    session,
    messages,
    memories,
    cwd,
    teleportedAt: Date.now(),
    sourceInstance: process.env.EUCLID_INSTANCE_ID || 'local',
  };

  fs.writeFileSync(path.join(bundleDir, 'session.json'), JSON.stringify(sessionState, null, 2));

  // Create final archive
  const archivePath = path.join(os.tmpdir(), `euclid-teleport-${sessionId.slice(0, 8)}.tar.gz`);
  execSync(`tar czf "${archivePath}" -C "${bundleDir}" .`);
  fs.rmSync(bundleDir, { recursive: true });

  return { archivePath, sessionId };
}

/**
 * Push a teleport bundle to a remote Euclid instance.
 */
async function pushTeleport(archivePath, targetUrl, apiKey) {
  const FormData = require('form-data');
  const form = new FormData();
  form.append('bundle', fs.createReadStream(archivePath));

  const res = await fetch(`${targetUrl}/api/sessions/import`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${apiKey}`,
      ...form.getHeaders(),
    },
    body: form,
  });

  const data = await res.json();
  fs.unlinkSync(archivePath);
  return data;
}

module.exports = { createTeleportBundle, pushTeleport };
```

Add `POST /api/sessions/:id/teleport` accepting `{ targetUrl, apiKey }`. Add `POST /api/sessions/import` accepting the bundle and importing the session state. Add `teleport_history` table to track teleportation events.

---

## Appendix AC — Cost Tracking and Effort Scoring

**Source**: `src/cost-tracker.ts`, `src/costHook.ts`, `src/utils/modelCost.ts`, `src/utils/effort.ts`, `src/components/EffortIndicator.ts`

### Cost Tracker

Every model call in Euclid should track token usage and estimated cost. The cost data accumulates across the session and is surfaced in the UI.

```javascript
// src/services/costTracker.js
'use strict';

const { db } = require('../db');

// Cost per million tokens for each model (USD)
// Source: src/utils/modelCost.ts adapted for open models (GPU rental cost estimate)
const MODEL_COSTS = {
  // Ollama on RunPod A100 (~$2.50/hr) — rough estimate: 2M tokens/hr
  'qwen2.5-coder:32b': { input: 0.00125, output: 0.00125 }, // $1.25/M tokens
  'qwen2.5-coder:7b': { input: 0.00030, output: 0.00030 },
  'qwen2.5:3b': { input: 0.00010, output: 0.00010 },
  'default': { input: 0.00100, output: 0.00100 },
};

function getCostForModel(modelName) {
  for (const [key, costs] of Object.entries(MODEL_COSTS)) {
    if (modelName.startsWith(key)) return costs;
  }
  return MODEL_COSTS.default;
}

/**
 * Record a model API call's token usage.
 */
function recordUsage(sessionId, model, promptTokens, completionTokens, durationMs) {
  const costs = getCostForModel(model);
  const inputCostUSD = (promptTokens / 1_000_000) * costs.input;
  const outputCostUSD = (completionTokens / 1_000_000) * costs.output;
  const totalCostUSD = inputCostUSD + outputCostUSD;

  db.prepare(`
    INSERT INTO cost_events (session_id, model, prompt_tokens, completion_tokens,
      input_cost_usd, output_cost_usd, total_cost_usd, duration_ms, created_at)
    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
  `).run(sessionId, model, promptTokens, completionTokens,
         inputCostUSD, outputCostUSD, totalCostUSD, durationMs, Date.now());

  return { inputCostUSD, outputCostUSD, totalCostUSD };
}

/**
 * Get total cost and usage for a session.
 */
function getSessionCost(sessionId) {
  return db.prepare(`
    SELECT
      SUM(prompt_tokens) as total_prompt_tokens,
      SUM(completion_tokens) as total_completion_tokens,
      SUM(total_cost_usd) as total_cost_usd,
      COUNT(*) as api_calls,
      AVG(duration_ms) as avg_duration_ms
    FROM cost_events WHERE session_id = ?
  `).get(sessionId);
}

/**
 * Get cost breakdown by model for a session.
 */
function getSessionCostByModel(sessionId) {
  return db.prepare(`
    SELECT model, SUM(total_cost_usd) as cost, SUM(prompt_tokens + completion_tokens) as tokens, COUNT(*) as calls
    FROM cost_events WHERE session_id = ?
    GROUP BY model ORDER BY cost DESC
  `).all(sessionId);
}

module.exports = { recordUsage, getSessionCost, getSessionCostByModel, getCostForModel };
```

Add the `cost_events` table to the schema (the original schema has `metrics` — add this as a specialized cost table):

```sql
CREATE TABLE IF NOT EXISTS cost_events (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id TEXT NOT NULL,
  model TEXT NOT NULL,
  prompt_tokens INTEGER DEFAULT 0,
  completion_tokens INTEGER DEFAULT 0,
  input_cost_usd REAL DEFAULT 0.0,
  output_cost_usd REAL DEFAULT 0.0,
  total_cost_usd REAL DEFAULT 0.0,
  duration_ms INTEGER DEFAULT 0,
  created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);
CREATE INDEX IF NOT EXISTS idx_cost_session ON cost_events(session_id, created_at DESC);
```

Wire `recordUsage` into the Ollama client's `complete()` and `stream()` functions — call it after every successful model response with the usage data from `parsed.usage`. Wire into the query loop's `cumulativeUsage` accumulator.

### Effort Scorer

The effort scorer from `src/utils/effort.ts` estimates how difficult a task is before starting it, which helps set appropriate `maxTurns` budgets.

```javascript
// src/utils/effortScorer.js
'use strict';

const { complete } = require('../services/ollama/client');
const { getModel } = require('./modelRouter');

// Effort levels matching Claude Code's effort.ts
const EFFORT_LEVELS = {
  TRIVIAL: { label: 'trivial', maxTurns: 5, description: 'Single-step change' },
  LOW: { label: 'low', maxTurns: 15, description: 'Simple bug fix or addition' },
  MEDIUM: { label: 'medium', maxTurns: 30, description: 'Feature implementation' },
  HIGH: { label: 'high', maxTurns: 60, description: 'Complex refactor or new system' },
  EXTREME: { label: 'extreme', maxTurns: 100, description: 'Architecture change or migration' },
};

/**
 * Estimate the effort level of a task from its description.
 * Returns an EFFORT_LEVELS entry.
 */
async function scoreEffort(taskDescription) {
  // Fast keyword-based classification first (avoid model call for obvious cases)
  const lower = taskDescription.toLowerCase();

  if (lower.match(/\b(fix typo|rename|add comment|update readme)\b/)) return EFFORT_LEVELS.TRIVIAL;
  if (lower.match(/\b(migrate|refactor|redesign|overhaul|rewrite)\b/)) return EFFORT_LEVELS.HIGH;
  if (lower.match(/\b(new service|new api|authentication|database schema)\b/)) return EFFORT_LEVELS.HIGH;

  // Use the compact model for ambiguous cases
  const result = await complete({
    model: getModel('compact'),
    messages: [{
      role: 'user',
      content: `Rate the engineering effort for this task. Reply with ONE WORD: trivial, low, medium, high, or extreme.

Task: ${taskDescription}`,
    }],
    options: { temperature: 0, num_predict: 10 },
  });

  const label = (result?.message?.content || 'medium').trim().toLowerCase();
  return EFFORT_LEVELS[label.toUpperCase()] || EFFORT_LEVELS.MEDIUM;
}

module.exports = { scoreEffort, EFFORT_LEVELS };
```

Wire `scoreEffort` into `QueryEngine.submitMessage()`: on the first turn of a session (turn 1), call it and use the result to set `maxTurns` if the user hasn't specified one. Surface the effort estimate in the SSE stream as `{ type: 'effort_estimate', level: 'medium', maxTurns: 30 }`.

---

## Appendix AD — Sandbox System

**Source**: `src/entrypoints/sandboxTypes.ts`, `src/utils/sandbox/`, `src/commands/sandbox-toggle/`, `src/components/sandbox/`

The sandbox system isolates the agent's file and network access using OS-level mechanisms. On macOS, it uses `sandbox-exec`. On Linux, it uses `seccomp` profiles or `bwrap` (Bubblewrap). The sandbox restricts: which directories the agent can write to, whether it can access the network, and which system calls it can make.

```javascript
// src/services/sandbox/sandboxAdapter.js
'use strict';

const { execSync, spawn } = require('child_process');

const SANDBOX_BACKENDS = {
  darwin: 'sandbox-exec',
  linux: 'bwrap',
};

// Detect available sandbox backend
const PLATFORM = process.platform;
const SANDBOX_BACKEND = process.env.EUCLID_SANDBOX_BACKEND ||
  (PLATFORM === 'darwin' && hasBinary('sandbox-exec') ? 'sandbox-exec' :
   PLATFORM === 'linux' && hasBinary('bwrap') ? 'bwrap' : 'none');

function hasBinary(name) {
  try { execSync(`which ${name}`, { stdio: 'ignore' }); return true; } catch { return false; }
}

const SANDBOX_PROFILES = {
  strict: {
    // Read-only filesystem except CWD; no network
    description: 'Read-only filesystem except CWD; no network access',
    allowNetwork: false,
    writeDirs: [], // Set from CWD at runtime
    readDirs: ['/', '/usr', '/etc', '/bin', '/sbin', '/lib'],
  },
  standard: {
    description: 'Write access to CWD and temp; no external network',
    allowNetwork: 'localhost',
    writeDirs: [],
    readDirs: null, // All readable
  },
  permissive: {
    description: 'Full write access; network allowed to approved hosts',
    allowNetwork: true,
    writeDirs: null,
    readDirs: null,
  },
};

/**
 * Wrap a command in the appropriate sandbox.
 * Returns { command, args } that can be passed to spawn().
 */
function sandboxWrapCommand(command, args, { cwd, profile = 'standard' } = {}) {
  const config = SANDBOX_PROFILES[profile] || SANDBOX_PROFILES.standard;

  if (SANDBOX_BACKEND === 'none') {
    return { command, args };
  }

  if (SANDBOX_BACKEND === 'sandbox-exec') {
    // macOS sandbox-exec profile
    const profileContent = buildMacOSSandboxProfile(config, cwd);
    const profileFile = `/tmp/euclid-sandbox-${Date.now()}.sb`;
    require('fs').writeFileSync(profileFile, profileContent);

    return {
      command: 'sandbox-exec',
      args: ['-f', profileFile, command, ...args],
      cleanup: () => { try { require('fs').unlinkSync(profileFile); } catch {} },
    };
  }

  if (SANDBOX_BACKEND === 'bwrap') {
    // Linux Bubblewrap
    const bwrapArgs = buildBwrapArgs(config, cwd);
    return {
      command: 'bwrap',
      args: [...bwrapArgs, '--', command, ...args],
    };
  }

  return { command, args };
}

function buildMacOSSandboxProfile(config, cwd) {
  const lines = ['(version 1)', '(deny default)'];
  lines.push('(allow process-exec*)');
  lines.push('(allow process-fork)');
  lines.push('(allow file-read* (subpath "/usr/lib"))');
  lines.push('(allow file-read* (subpath "/System"))');
  if (cwd) lines.push(`(allow file-write* (subpath "${cwd}"))`);
  lines.push('(allow file-write* (subpath "/tmp"))');
  if (config.allowNetwork === true) lines.push('(allow network*)');
  if (config.allowNetwork === 'localhost') lines.push('(allow network-outbound (local ip))');
  return lines.join('\n');
}

function buildBwrapArgs(config, cwd) {
  const args = [
    '--unshare-pid',
    '--ro-bind', '/', '/',
    '--dev', '/dev',
    '--proc', '/proc',
    '--tmpfs', '/tmp',
  ];
  if (cwd) args.push('--bind', cwd, cwd);
  if (!config.allowNetwork) args.push('--unshare-net');
  return args;
}

module.exports = { sandboxWrapCommand, SANDBOX_BACKEND, SANDBOX_PROFILES };
```

The sandbox is applied in the `BashTool` — when `EUCLID_SANDBOX=true` and the tool is executing an untrusted command (safety score > 40), wrap it with `sandboxWrapCommand`. Add `EUCLID_SANDBOX=false` and `EUCLID_SANDBOX_PROFILE=standard` to `.env.example`. Install bubblewrap on Linux: `apt-get install -y bubblewrap`.

---

## Appendix AE — Rate Limiting and Policy Limits

**Source**: `src/services/rateLimitMessages.ts`, `src/services/policyLimits/`, `src/commands/rate-limit-options/`

Policy limits define maximum resource consumption per session or per time window. This prevents a runaway agent from consuming excessive GPU time, making too many API calls, or writing too many files.

```javascript
// src/services/policyLimits/index.js
'use strict';

const { db } = require('../../db');

// Default policy limits — can be overridden per session via prefs
const DEFAULT_LIMITS = {
  maxTurnsPerSession: 200,
  maxTokensPerSession: 2_000_000,
  maxToolCallsPerTurn: 20,
  maxBashTimeoutSeconds: 300,
  maxFileSizeBytes: 10 * 1024 * 1024, // 10MB
  maxContextWindowChars: 500_000,
  maxConcurrentSessions: 5,
  rateLimitRequestsPerMinute: 60,
};

function getSessionLimits(sessionId) {
  const override = db.prepare("SELECT value FROM prefs WHERE key = 'policy.limits'").get();
  return override ? { ...DEFAULT_LIMITS, ...JSON.parse(override.value) } : DEFAULT_LIMITS;
}

function checkTurnLimit(sessionId, currentTurn) {
  const limits = getSessionLimits(sessionId);
  if (currentTurn > limits.maxTurnsPerSession) {
    return {
      exceeded: true,
      message: `Turn limit reached (${limits.maxTurnsPerSession}). Start a new session or increase maxTurnsPerSession.`,
    };
  }
  return { exceeded: false };
}

function checkTokenLimit(sessionId, totalTokens) {
  const limits = getSessionLimits(sessionId);
  if (totalTokens > limits.maxTokensPerSession) {
    return {
      exceeded: true,
      message: `Token limit reached (${limits.maxTokensPerSession.toLocaleString()}). Consider compacting or starting a new session.`,
    };
  }

  const pct = Math.round((totalTokens / limits.maxTokensPerSession) * 100);
  if (pct > 80) {
    return {
      exceeded: false,
      warning: `Token usage at ${pct}% of session limit.`,
    };
  }

  return { exceeded: false };
}

/**
 * Rate limit tracker for API calls.
 * Uses a sliding window counter in the DB.
 */
function checkRateLimit(sessionId, windowSeconds = 60) {
  const limits = getSessionLimits(sessionId);
  const windowStart = Date.now() - windowSeconds * 1000;

  const count = db.prepare(`
    SELECT COUNT(*) as count FROM cost_events
    WHERE session_id = ? AND created_at > ?
  `).get(sessionId, windowStart)?.count || 0;

  if (count >= limits.rateLimitRequestsPerMinute) {
    const retryAfterMs = windowSeconds * 1000 - (Date.now() - windowStart);
    return {
      limited: true,
      retryAfterMs,
      message: `Rate limit: ${limits.rateLimitRequestsPerMinute} requests/${windowSeconds}s. Retry in ${Math.ceil(retryAfterMs / 1000)}s.`,
    };
  }

  return { limited: false };
}

module.exports = { getSessionLimits, checkTurnLimit, checkTokenLimit, checkRateLimit, DEFAULT_LIMITS };
```

Wire all three checks into `queryLoop.js`:
- `checkTurnLimit` at the top of each iteration
- `checkTokenLimit` after updating `cumulativeUsage`
- `checkRateLimit` before each model call in `complete()` or `stream()`

---

## Appendix AF — Agent Built-in Specialist Definitions

**Source**: `src/tools/AgentTool/built-in/` — `claudeCodeGuideAgent.ts`, `exploreAgent.ts`, `generalPurposeAgent.ts`, `planAgent.ts`, `verificationAgent.ts`, `statuslineSetup.ts`

These are the **built-in sub-agent definitions** from the Claude Code source — specialist agents that are always available without needing a custom agent definition. Euclid replicates all five.

```javascript
// src/agents/builtins.js — EXTEND with these 5 additional built-ins
const ADDITIONAL_BUILTIN_AGENTS = [
  {
    id: 'explorer',
    name: 'Explorer',
    type: 'builtin',
    temperature: 0.2,
    system_prompt: `You are an expert code explorer. Your job is to rapidly understand an unfamiliar codebase.
When asked to explore, you:
1. Read the directory structure with ls/glob
2. Find entry points: package.json, main.ts, index.js, README.md
3. Trace the call graph from entry points to understand data flow
4. Identify the key files (top 10) that matter most
5. Produce a structured summary: architecture, key files, data flow, and any surprises
You do NOT make changes — you only read and report. Speed is paramount.`,
  },
  {
    id: 'guide',
    name: 'Guide',
    type: 'builtin',
    temperature: 0.3,
    system_prompt: `You are a coding guide that helps users understand how to use Euclid effectively.
You know all of Euclid's capabilities: tools, slash commands, memory system, multi-agent features, skill system, and configuration.
When a user asks how to do something, you give a concrete, actionable answer with examples.
You do not make code changes — you explain and demonstrate.`,
  },
  {
    id: 'verifier',
    name: 'Verifier',
    type: 'builtin',
    temperature: 0.1,
    system_prompt: `You are a rigorous verification agent. Your job is to verify that code changes are correct.
For every change you verify:
1. Read the modified files
2. Run tests: npm test, pytest, go test, etc.
3. Run linting: eslint, pylint, gofmt, etc.
4. Check for obvious bugs: null derefs, off-by-one, missing error handling
5. Verify the change matches the stated intent
6. Report: PASS (all clear) or FAIL (list specific issues)
You do not fix bugs you find — you report them for the Coder to fix.`,
  },
  {
    id: 'plan-agent',
    name: 'Plan Agent',
    type: 'builtin',
    temperature: 0.2,
    system_prompt: `You are a planning specialist. You produce detailed implementation plans.
Your plans include:
1. Pre-conditions: what must be true before work begins
2. Ordered steps: each step is atomic and verifiable
3. Dependencies between steps (what blocks what)
4. Risk factors: what could go wrong at each step
5. Success criteria: how to know the task is done
Plans must be executable by another agent without additional clarification.
You use plan mode: explore and read before proposing any changes.`,
  },
  {
    id: 'general',
    name: 'General',
    type: 'builtin',
    temperature: 0.3,
    system_prompt: `You are a general-purpose AI assistant with access to a full coding environment.
You can read files, write code, run commands, search the web, manage memory, and coordinate with specialist agents.
Balance thoroughness with efficiency. Always verify your work. Be direct about what you are doing and why.`,
  },
];

// Merge into the existing BUILTIN_AGENTS array in builtins.js
module.exports = { ADDITIONAL_BUILTIN_AGENTS };
```

---

## Appendix AG — Memory System: findRelevantMemories, memoryAge, memoryScan

**Source**: `src/memdir/` — `findRelevantMemories.ts`, `memdir.ts`, `memoryAge.ts`, `memoryScan.ts`, `memoryTypes.ts`, `paths.ts`

The `memdir` system is how Claude Code manages memory files on disk. Memory is not stored only in SQLite — it is also written to `~/.claude/projects/<hash>/memory/*.md` files. The `memoryScan` function discovers all memory files, `memoryAge` scores them by relevance decay, and `findRelevantMemories` returns the best memories for the current session context.

```javascript
// src/services/memory/memoryScan.js
'use strict';

const fs = require('fs');
const path = require('path');
const crypto = require('crypto');
const os = require('os');

// Memory types (matching Claude Code's memoryTypes.ts)
const MEMORY_TYPES = {
  AUTO: 'auto',      // Automatically extracted from session
  USER: 'user',      // Manually added by user ("remember X")
  TEAM: 'team',      // Shared across sessions/users
  PROJECT: 'project', // Project-wide facts
};

function getProjectMemoryDir(cwd) {
  const hash = crypto.createHash('sha256').update(cwd || process.cwd()).digest('hex').slice(0, 16);
  return path.join(os.homedir(), '.euclid', 'projects', hash, 'memory');
}

/**
 * Scan all memory files for a project and return them with age metadata.
 * Source: memoryScan.ts
 */
function scanMemoryFiles(cwd) {
  const memDir = getProjectMemoryDir(cwd);
  if (!fs.existsSync(memDir)) return [];

  const files = fs.readdirSync(memDir)
    .filter(f => f.endsWith('.json') || f.endsWith('.md'))
    .sort()
    .reverse(); // Newest first

  return files.map(file => {
    const filePath = path.join(memDir, file);
    const stat = fs.statSync(filePath);
    const ageMs = Date.now() - stat.mtimeMs;

    let content = '';
    try {
      const raw = fs.readFileSync(filePath, 'utf8');
      content = file.endsWith('.json') ? JSON.stringify(JSON.parse(raw), null, 2) : raw;
    } catch { content = ''; }

    return {
      file,
      path: filePath,
      content,
      ageMs,
      ageDays: ageMs / (1000 * 60 * 60 * 24),
      type: file.includes('auto') ? MEMORY_TYPES.AUTO :
            file.includes('user') ? MEMORY_TYPES.USER :
            file.includes('team') ? MEMORY_TYPES.TEAM : MEMORY_TYPES.PROJECT,
    };
  });
}

/**
 * Score memories by relevance, combining recency and content match.
 * Source: findRelevantMemories.ts
 */
function findRelevantMemories(cwd, query, { maxAge = 30, topK = 10 } = {}) {
  const memories = scanMemoryFiles(cwd);
  if (!memories.length) return [];

  // Filter by age (days)
  const fresh = memories.filter(m => m.ageDays <= maxAge);

  if (!query) {
    // No query — return freshest memories by type priority
    return fresh
      .sort((a, b) => {
        // User memories first, then auto, then project
        const priority = { user: 0, project: 1, auto: 2, team: 3 };
        return (priority[a.type] || 9) - (priority[b.type] || 9) || a.ageMs - b.ageMs;
      })
      .slice(0, topK);
  }

  // BM25-based relevance scoring using the query
  const queryTokens = tokenize(query);

  const scored = fresh.map(m => {
    const contentTokens = tokenize(m.content);
    let score = 0;
    for (const qt of queryTokens) {
      if (contentTokens.includes(qt)) score += 2;
      if (m.content.toLowerCase().includes(qt.toLowerCase())) score += 1;
    }
    // Recency boost: newer memories score higher
    const recencyBoost = Math.max(0, 1 - m.ageDays / maxAge);
    score += recencyBoost;
    return { ...m, score };
  });

  return scored
    .filter(m => m.score > 0)
    .sort((a, b) => b.score - a.score)
    .slice(0, topK);
}

/**
 * Format memory manifest for system prompt injection.
 * Source: memoryScan.ts formatMemoryManifest
 */
function formatMemoryManifest(memories) {
  if (!memories.length) return '';

  const sections = memories.reduce((acc, m) => {
    const key = m.type;
    if (!acc[key]) acc[key] = [];
    acc[key].push(m.content.slice(0, 300));
    return acc;
  }, {});

  const parts = [];
  if (sections.user) parts.push(`## User Preferences\n${sections.user.join('\n')}`);
  if (sections.project) parts.push(`## Project Facts\n${sections.project.join('\n')}`);
  if (sections.auto) parts.push(`## Session History\n${sections.auto.join('\n')}`);
  if (sections.team) parts.push(`## Team Knowledge\n${sections.team.join('\n')}`);

  return parts.join('\n\n');
}

function tokenize(text) {
  return (text || '').toLowerCase().split(/\W+/).filter(t => t.length > 2);
}

module.exports = {
  scanMemoryFiles,
  findRelevantMemories,
  formatMemoryManifest,
  getProjectMemoryDir,
  MEMORY_TYPES,
};
```

Wire `findRelevantMemories` into `QueryEngine.submitMessage()`: on every first-turn session start, call it with the user's message as the query and inject the `formatMemoryManifest` result into the system prompt's memory section.

---

## Appendix AH — Hooks System Deep Dive: Agent, HTTP, and Prompt Hook Types

**Source**: `src/utils/hooks/execAgentHook.ts`, `src/utils/hooks/execHttpHook.ts`, `src/utils/hooks/execPromptHook.ts`, `src/utils/hooks/AsyncHookRegistry.ts`, `src/utils/hooks/ssrfGuard.ts`

Beyond the shell-based hooks in Appendix G, Claude Code supports three additional hook execution types: **agent hooks** (run a sub-agent), **HTTP hooks** (call an external webhook), and **prompt hooks** (inject model-generated content). These enable advanced automation pipelines.

```javascript
// src/services/hooks/hookTypes.js
'use strict';

const { complete } = require('../ollama/client');
const { getModel } = require('../../utils/modelRouter');

// SSRF Guard: block hooks from calling internal/private network addresses
// Source: src/utils/hooks/ssrfGuard.ts
const BLOCKED_RANGES = [
  /^(10\.|192\.168\.|172\.(1[6-9]|2\d|3[01])\.)/,  // Private ranges
  /^127\./,                                           // Loopback
  /^169\.254\./,                                      // Link-local
  /localhost/i,
  /0\.0\.0\.0/,
];

function ssrfGuard(url) {
  try {
    const parsed = new URL(url);
    const host = parsed.hostname;
    for (const pattern of BLOCKED_RANGES) {
      if (pattern.test(host)) {
        // Allow localhost if EUCLID_ALLOW_LOCALHOST_HOOKS=true
        if (host === 'localhost' || host === '127.0.0.1') {
          if (process.env.EUCLID_ALLOW_LOCALHOST_HOOKS === 'true') return true;
        }
        return false;
      }
    }
    return true;
  } catch { return false; }
}

/**
 * Execute an HTTP hook — POST to an external webhook URL.
 * Source: execHttpHook.ts
 */
async function execHttpHook(hookConfig, hookEnv, toolResult) {
  const url = hookConfig.url;
  if (!url) throw new Error('HTTP hook missing url');

  if (!ssrfGuard(url)) {
    throw new Error(`HTTP hook URL blocked by SSRF guard: ${url}`);
  }

  const payload = {
    event: hookEnv.EUCLID_HOOK_TYPE,
    sessionId: hookEnv.EUCLID_SESSION_ID,
    toolName: hookEnv.EUCLID_TOOL_NAME,
    toolInput: hookEnv.EUCLID_TOOL_INPUT ? JSON.parse(hookEnv.EUCLID_TOOL_INPUT) : null,
    toolResult: toolResult || null,
    timestamp: Date.now(),
  };

  const res = await fetch(url, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-Euclid-Hook': '1',
      ...(hookConfig.headers || {}),
    },
    body: JSON.stringify(payload),
    signal: AbortSignal.timeout(hookConfig.timeout_ms || 10_000),
  });

  if (!res.ok) {
    throw new Error(`HTTP hook failed: ${res.status} ${res.statusText}`);
  }

  const responseText = await res.text();
  try { return JSON.parse(responseText); } catch { return { response: responseText }; }
}

/**
 * Execute an agent hook — run a sub-agent model call that produces hook output.
 * Source: execAgentHook.ts
 */
async function execAgentHook(hookConfig, hookEnv, context) {
  const systemPrompt = hookConfig.system_prompt ||
    'You are a hook processor. Analyze the provided context and return a JSON response.';

  const userContent = [
    `Hook event: ${hookEnv.EUCLID_HOOK_TYPE}`,
    hookEnv.EUCLID_TOOL_NAME ? `Tool: ${hookEnv.EUCLID_TOOL_NAME}` : '',
    hookEnv.EUCLID_TOOL_INPUT ? `Input: ${hookEnv.EUCLID_TOOL_INPUT}` : '',
    hookConfig.prompt || 'Process this hook event.',
  ].filter(Boolean).join('\n\n');

  const result = await complete({
    model: hookConfig.model || getModel('compact'),
    messages: [
      { role: 'system', content: systemPrompt },
      { role: 'user', content: userContent },
    ],
    options: { temperature: 0.1, num_predict: 1024 },
  });

  const content = result?.message?.content || '';
  try {
    return JSON.parse(content.replace(/```json\s*/g, '').replace(/```/g, '').trim());
  } catch {
    return { output: content };
  }
}

/**
 * Execute a prompt hook — inject model-generated content into the tool result.
 * Source: execPromptHook.ts
 */
async function execPromptHook(hookConfig, toolName, toolInput, toolResult) {
  if (!hookConfig.prompt) return toolResult;

  const result = await complete({
    model: hookConfig.model || getModel('compact'),
    messages: [{
      role: 'user',
      content: `${hookConfig.prompt}

Tool: ${toolName}
Input: ${JSON.stringify(toolInput)}
Result: ${JSON.stringify(toolResult)}

Output ONLY valid JSON with the modified result, or the original if no changes needed.`,
    }],
    options: { temperature: 0.1, num_predict: 512 },
  });

  try {
    return JSON.parse(result?.message?.content || '');
  } catch {
    return toolResult; // Return original if parsing fails
  }
}

/**
 * AsyncHookRegistry: manages hooks that run asynchronously (fire-and-forget).
 * Source: AsyncHookRegistry.ts
 */
class AsyncHookRegistry {
  constructor() {
    this.pending = new Set();
    this.maxPending = 50;
  }

  async fire(hookFn, ...args) {
    if (this.pending.size >= this.maxPending) {
      console.warn('[hooks] AsyncHookRegistry at capacity, dropping hook');
      return;
    }

    const promise = hookFn(...args)
      .catch(err => console.error('[hooks] Async hook error:', err.message))
      .finally(() => this.pending.delete(promise));

    this.pending.add(promise);
  }

  async drain() {
    await Promise.allSettled([...this.pending]);
  }
}

const asyncHookRegistry = new AsyncHookRegistry();

module.exports = { execHttpHook, execAgentHook, execPromptHook, ssrfGuard, asyncHookRegistry };
```

Update the hook config format in `.euclid/hooks.json` to support the new hook types:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "tool_name": "file_write" },
        "type": "http",
        "url": "https://your-webhook.example.com/file-changed",
        "timeout_ms": 5000
      },
      {
        "matcher": { "tool_name": "bash" },
        "type": "agent",
        "model": "qwen2.5:3b",
        "system_prompt": "Analyze this bash command for security issues.",
        "prompt": "Did this bash command pose any security risks? Reply with JSON { risk: low|medium|high, reason: string }."
      },
      {
        "matcher": null,
        "type": "prompt",
        "prompt": "Improve the tool result format for readability.",
        "model": "qwen2.5:3b"
      }
    ]
  }
}
```

Update `runPostToolUseHooks` in `src/services/hooks/hookRunner.js` to dispatch to `execHttpHook`, `execAgentHook`, or `execPromptHook` based on `hook.type`.

---

## Appendix AI — Agent Memory and Snapshot System

**Source**: `src/tools/AgentTool/agentMemory.ts`, `src/tools/AgentTool/agentMemorySnapshot.ts`

Agent memory is separate from session memory — each sub-agent can have its own private memory namespace. When a sub-agent completes its task, its memory is optionally persisted as a snapshot that can be used to restore the agent's context in a future invocation (without re-reading all the same files).

```javascript
// src/services/agents/agentMemory.js
'use strict';

const { db } = require('../../db');
const { v4: uuidv4 } = require('uuid');

/**
 * Agent memory operations — per-agent namespace in the memory table.
 */
function getAgentMemory(agentId) {
  return db.prepare("SELECT * FROM memory WHERE namespace = ? ORDER BY created_at DESC LIMIT 50")
    .all(`agent:${agentId}`);
}

function setAgentMemory(agentId, key, value) {
  const existing = db.prepare(
    "SELECT id FROM memory WHERE namespace = ? AND content LIKE ?"
  ).get(`agent:${agentId}`, `${key}:%`);

  if (existing) {
    db.prepare("UPDATE memory SET content = ?, created_at = ? WHERE id = ?")
      .run(`${key}: ${value}`, Date.now(), existing.id);
  } else {
    db.prepare("INSERT INTO memory (id, namespace, content, created_at) VALUES (?, ?, ?, ?)")
      .run(uuidv4(), `agent:${agentId}`, `${key}: ${value}`, Date.now());
  }
}

/**
 * Create a memory snapshot of an agent's state.
 * This is stored when the agent task completes, enabling warm-resume.
 * Source: agentMemorySnapshot.ts
 */
function snapshotAgentMemory(agentId, { taskSummary, filesRead, filesWritten, keyFacts }) {
  const snapshotId = uuidv4();
  const snapshot = {
    agentId,
    snapshotId,
    taskSummary,
    filesRead: filesRead || [],
    filesWritten: filesWritten || [],
    keyFacts: keyFacts || [],
    createdAt: Date.now(),
  };

  db.prepare(`
    INSERT INTO agent_memory_snapshots (id, agent_id, snapshot, created_at)
    VALUES (?, ?, ?, ?)
  `).run(snapshotId, agentId, JSON.stringify(snapshot), Date.now());

  return snapshotId;
}

/**
 * Load a snapshot to warm-resume an agent.
 */
function loadAgentSnapshot(agentId) {
  const row = db.prepare(`
    SELECT snapshot FROM agent_memory_snapshots WHERE agent_id = ? ORDER BY created_at DESC LIMIT 1
  `).get(agentId);

  return row ? JSON.parse(row.snapshot) : null;
}

/**
 * Format a snapshot as system prompt addendum for warm-resume.
 */
function formatSnapshotForPrompt(snapshot) {
  if (!snapshot) return '';

  return `## Previous Agent Session
Task: ${snapshot.taskSummary}
Files read: ${snapshot.filesRead.slice(0, 10).join(', ')}
Files written: ${snapshot.filesWritten.slice(0, 5).join(', ')}
Key facts established:
${snapshot.keyFacts.map(f => `- ${f}`).join('\n')}

Resume from this context — do not re-read files you've already processed.`;
}

module.exports = { getAgentMemory, setAgentMemory, snapshotAgentMemory, loadAgentSnapshot, formatSnapshotForPrompt };
```

Add `agent_memory_snapshots` table:

```sql
CREATE TABLE IF NOT EXISTS agent_memory_snapshots (
  id TEXT PRIMARY KEY,
  agent_id TEXT NOT NULL,
  snapshot TEXT NOT NULL,
  created_at INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);
CREATE INDEX IF NOT EXISTS idx_agent_snapshots ON agent_memory_snapshots(agent_id, created_at DESC);
```

---

## Appendix AJ — UltraPlan and Thinkback

**Source**: `src/commands/ultraplan.tsx`, `src/utils/ultraplan/`, `src/commands/thinkback/`, `src/commands/thinkback-play/`

### UltraPlan

UltraPlan is an extended version of Plan Mode that uses a **CCR (Cloud Compute Resource) session** for the planning phase — a separate long-running model call that does deep exploration before producing the plan. The plan is more comprehensive because it has its own dedicated context window, separate from the main conversation.

```javascript
// src/services/plan/ultraPlan.js
'use strict';

const { stream } = require('../ollama/client');
const { getModel } = require('../../utils/modelRouter');
const { db } = require('../../db');
const { v4: uuidv4 } = require('uuid');

const ULTRAPLAN_SYSTEM_PROMPT = `You are an expert software architect in planning mode.
Your job is to produce a comprehensive, actionable implementation plan.

PLANNING PROCESS:
1. EXPLORE: Read all relevant files and understand the current state
2. ANALYZE: Identify constraints, risks, and dependencies  
3. DESIGN: Choose the best approach from multiple options
4. PLAN: Write a detailed numbered plan with:
   - Each step as a single atomic action
   - Expected output for each step
   - Rollback strategy for risky steps
   - Test to verify each step succeeded

DO NOT write code. DO NOT make changes. Read and plan only.
When your plan is complete, output: [PLAN COMPLETE]`;

/**
 * Run UltraPlan: a dedicated planning session separate from the main conversation.
 * Uses a fresh context window for deep exploration.
 */
async function runUltraPlan(task, { sessionId, cwd, model, signal, onChunk }) {
  const planId = uuidv4();
  const messages = [{ role: 'user', content: `Plan how to implement: ${task}` }];

  const tools = require('../../tools').loadTools({
    sessionId: `plan-${planId}`,
    permissionMode: 'readonly', // Plan mode is always readonly
    cwd,
  });

  let fullPlan = '';

  for await (const chunk of stream({
    model: model || getModel('main'),
    messages: [{ role: 'system', content: ULTRAPLAN_SYSTEM_PROMPT }, ...messages],
    tools: buildOllamaToolList(tools, { permissionMode: 'readonly' }),
    options: { temperature: 0.1, num_predict: 16384, num_ctx: 65536 },
  }, signal)) {
    if (chunk.type === 'text_delta') {
      fullPlan += chunk.delta;
      if (onChunk) onChunk(chunk.delta);
    }
    if (chunk.type === 'done') break;
  }

  // Store the plan
  db.prepare(`
    INSERT INTO plans (id, session_id, task, content, created_at)
    VALUES (?, ?, ?, ?, ?)
  `).run(planId, sessionId, task, fullPlan, Date.now());

  return { planId, plan: fullPlan };
}

function buildOllamaToolList(tools) {
  return tools
    .filter(t => t.isReadOnly && t.isReadOnly({}))
    .map(t => ({
      type: 'function',
      function: {
        name: t.name,
        description: t.description,
        parameters: t.inputSchema,
      },
    }));
}

module.exports = { runUltraPlan };
```

### Thinkback (Session Replay)

Thinkback allows replaying a past session to understand what the agent did and why. It renders the session's tool calls and model responses in a slow, animated replay.

```javascript
// src/services/thinkback/thinkback.js
'use strict';

const { db } = require('../../db');

/**
 * Load a session's full trace for replay.
 */
function loadSessionTrace(sessionId) {
  const messages = db.prepare(`
    SELECT id, role, content, created_at FROM messages
    WHERE session_id = ? ORDER BY created_at ASC
  `).all(sessionId);

  const toolCalls = db.prepare(`
    SELECT * FROM tool_calls WHERE session_id = ? ORDER BY created_at ASC
  `).all(sessionId);

  // Interleave messages and tool calls in chronological order
  const events = [
    ...messages.map(m => ({ ...m, _type: 'message' })),
    ...toolCalls.map(t => ({ ...t, _type: 'tool_call' })),
  ].sort((a, b) => a.created_at - b.created_at);

  return {
    sessionId,
    events,
    summary: {
      messageCount: messages.length,
      toolCallCount: toolCalls.length,
      duration: messages.length > 1
        ? messages[messages.length - 1].created_at - messages[0].created_at
        : 0,
    },
  };
}

/**
 * Stream a session replay, one event at a time, with configurable speed.
 * Returns an async generator of replay events.
 */
async function* streamReplay(sessionId, { speedMultiplier = 1.0 } = {}) {
  const trace = loadSessionTrace(sessionId);

  yield { type: 'replay_start', ...trace.summary };

  let prevTime = trace.events[0]?.created_at || Date.now();

  for (const event of trace.events) {
    const delay = Math.min(
      (event.created_at - prevTime) / speedMultiplier,
      3000 // Cap delay at 3 seconds for long pauses
    );

    await new Promise(r => setTimeout(r, delay));
    prevTime = event.created_at;

    yield { type: 'replay_event', event };
  }

  yield { type: 'replay_complete' };
}

module.exports = { loadSessionTrace, streamReplay };
```

Add `GET /api/sessions/:id/trace` and `GET /api/sessions/:id/replay` (SSE stream). Add `POST /api/plans` for saving ultraplans. Register `POST /api/ultraplan` route that calls `runUltraPlan` and streams the result.

---

## Appendix AK — Complete Updated Project Scaffold

The project scaffold from Section 4 and Appendix P must be further expanded to include all new subsystems:

```bash
# Complete scaffold — run in order
mkdir -p euclid/{src,public,data,uploads,logs,skills,agents,evals,scripts,hooks,worktrees,plugins}

# Source subdirectories
mkdir -p euclid/src/{db,utils,tools,routes,services,middleware,agents,workers,commands}

# Tools — all 32 tool directories
mkdir -p euclid/src/tools/{bash,read,write,edit,glob,grep,ls,agent,mcp,search,todo,\
askUser,toolSearch,webFetch,webSearch,sleep,sendMessage,cron,lsp,notebook,repl,\
taskTools,plan,worktree,computerUse,brief,config,teams,swarm}

# Services — all service directories
mkdir -p euclid/src/services/{ollama,compact,rag,skill,task,permission,hooks,cron,\
memory,bridge,swarm,voice,computerUse,autoDream,plan,sandbox,costTracker,\
policyLimits,agentMemory,thinkback,plugins,session}

# Utils — all utility directories  
mkdir -p euclid/src/utils/{permissions,model,settings,bash,shell,swarm,hooks,\
mcp,plugins,memory,task,computerUse}

# Skills — all 29 external + bundled
mkdir -p euclid/src/skills/bundled
mkdir -p euclid/skills/{api-design,backend-patterns,coding-standards,e2e-testing,\
frontend-patterns,documentation-lookup,eval-harness,security-review,tdd-workflow,\
verification-loop,mcp-server-patterns,deep-research,strategic-compact,bun-runtime,\
claude-api,exa-search,fal-ai-media,x-api,article-writing,content-engine,crosspost,\
video-editing,frontend-slides,market-research,investor-materials,investor-outreach,\
nextjs-turbopack,dmux-workflows,everything-euclid}

# Workers
mkdir -p euclid/src/workers

cd euclid

# All npm packages required by all appendices
npm install express better-sqlite3 ollama node-fetch@2 multer helmet \
  express-rate-limit uuid crypto zod morgan compression ws node-cron \
  @modelcontextprotocol/sdk vscode-languageserver-types form-data

npm install --save-dev nodemon
```

### Final Updated package.json scripts

```json
{
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js",
    "dev:debug": "NODE_OPTIONS='--inspect' nodemon src/server.js",
    "db:reset": "node -e \"require('./src/db').reset()\"",
    "db:migrate": "node src/db/migrate.js",
    "db:backup": "cp data/euclid.db data/euclid.$(date +%Y%m%d%H%M%S).db",
    "test": "node --test src/**/*.test.js",
    "test:tools": "node --test src/tools/**/*.test.js",
    "hooks:validate": "node src/services/hooks/validate.js",
    "mcp:list": "node -e \"require('./src/services/mcp/client').initializeMcpServers(process.cwd()).then(r => console.log(JSON.stringify(r, null, 2)))\"",
    "memory:clear": "node -e \"require('./src/db').prepare('DELETE FROM memory').run()\"",
    "memory:autodream": "node -e \"require('./src/services/autoDream/consolidation').runConsolidationPass(null).then(console.log)\"",
    "swarm:kill-all": "tmux kill-server 2>/dev/null || true",
    "plugins:reload": "curl -X POST http://localhost:3000/api/plugins/reload",
    "evals:run": "node evals/runner.js",
    "sandbox:check": "node -e \"console.log(require('./src/services/sandbox/sandboxAdapter').SANDBOX_BACKEND)\"",
    "voice:check": "node -e \"console.log(require('./src/services/voice/stt').STT_BACKEND)\""
  }
}
```

### Final .env.example (Complete)

```bash
# Server
PORT=3000
NODE_ENV=development
AUTH_TOKEN=your-secret-token-here
BASE_URL=http://localhost:3000
EUCLID_INSTANCE_ID=local-1

# Model Configuration
EUCLID_MODEL=qwen2.5-coder:32b-instruct-q4_K_M
EUCLID_COMPACT_MODEL=qwen2.5:3b
EUCLID_PLANNER_MODEL=qwen2.5-coder:7b-instruct-q4_K_M
EUCLID_REFLEXION_MODEL=
OLLAMA_BASE_URL=http://localhost:11434
USE_OPENAI_COMPAT=false

# Inference Limits
EUCLID_MAX_TURNS=50
EUCLID_MAX_CONTEXT=32768

# Feature Gates
EUCLID_STREAMING_TOOLS=true
EUCLID_EMIT_TOOL_SUMMARIES=false
EUCLID_MEMORY_EXTRACT=true
EUCLID_HOOKS=true
EUCLID_LSP=false
EUCLID_REFLEXION=true
EUCLID_PLAN_MODE=true
EUCLID_AUTODREAM=true
EUCLID_AUTODREAM_INTERVAL_MS=86400000
EUCLID_ALLOW_BYPASS=false
EUCLID_FILE_HISTORY=true
EUCLID_COMPUTER_USE=false

# Voice
EUCLID_STT_BACKEND=auto
OPENAI_API_KEY=

# Search
EUCLID_SEARCH_BACKEND=searxng
SEARXNG_URL=http://localhost:8080
BRAVE_SEARCH_API_KEY=

# Sandbox
EUCLID_SANDBOX=false
EUCLID_SANDBOX_PROFILE=standard
EUCLID_ALLOW_LOCALHOST_HOOKS=false

# Remote/Bridge
EUCLID_BRIDGE_ENABLED=true
EUCLID_BRIDGE_SECRET_KEY=change-this-in-production

# Cost Tracking
EUCLID_COST_TRACKING=true
EUCLID_COST_ALERT_USD=5.00

# Database
DB_PATH=./data/euclid.db
```

---

*Euclid Volumes I and II now cover the complete Claude Code source: 1,902 TypeScript files across 37 directories, all 29 external skills plus 16 bundled skills, 32 tool implementations, 5 built-in agent specialists, the swarm system with 3 backends, the bridge remote-control layer, voice mode, computer use, AutoDream consolidation, streaming tool execution, plugin architecture, full permission depth with YOLO/bash classifiers, 3 hook execution types, the memory directory system, context analysis, microcompact, session backgrounding, teleport, cost tracking, effort scoring, the sandbox system, rate limiting, UltraPlan, and Thinkback. Every system is adapted for Ollama-based inference. Build it all.*
