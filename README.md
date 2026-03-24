# autodebug

Telemetry-Driven Inference Troubleshooting and Optimization Loop

Deploy a model as an inference service via [dstack](https://dstack.ai/), continuously fetch profiling telemetry from [Graphsignal](https://graphsignal.com/), analyze performance, and redeploy with improved configuration — all autonomously.

Inspired by [karpathy/autoresearch](https://github.com/karpathy/autoresearch). Where autoresearch lets an AI agent iterate on training code overnight, autodebug lets an AI agent iterate on inference deployment configuration: tuning batch sizes, caching strategies, parallelism, and engine parameters to minimize latency and maximize throughput.

## How it works

The agent follows an optimization loop defined in `program.md`:

1. **Deploy** an inference service (e.g. SGLang, vLLM) on dstack with Graphsignal telemetry enabled.
2. **Benchmark** the endpoint with targeted request patterns (parallel, sequential, long prompts, etc.).
3. **Fetch telemetry** from Graphsignal — profiling data, traces, metrics, and errors.
4. **Analyze** performance: compute prefill throughput, decode throughput, token throughput, and identify bottlenecks.
5. **Redeploy** with an optimized dstack configuration reflecting the improvements.
6. **Repeat** indefinitely.

Each iteration is logged to a separate `sessions/debug-<ISO>.md` file (findings and rationale), building a complete record of the optimization journey.

## Prerequisites

### 1. Graphsignal

[Graphsignal](https://graphsignal.com/) provides inference observability — profiling, tracing, and metrics. Sign up and obtain an API key.

Install the debug CLI and log in:

```bash
uv tool install graphsignal-debug
graphsignal-debug login
```

### 2. dstack

[dstack](https://dstack.ai/) manages cloud infrastructure for inference services. Set up your dstack project and log in:

```bash
uv tool install dstack
dstack login
```

### 3. Agent skills

Download the skill files so the agent has full context:

```bash
mkdir -p ~/.claude/skills/graphsignal-python ~/.claude/skills/graphsignal-debug ~/.claude/skills/dstack
curl -sL https://raw.githubusercontent.com/graphsignal/graphsignal-python/main/SKILL.md -o ~/.claude/skills/graphsignal-python/SKILL.md
curl -sL https://raw.githubusercontent.com/graphsignal/graphsignal-debug/main/SKILL.md -o ~/.claude/skills/graphsignal-debug/SKILL.md
curl -sL https://raw.githubusercontent.com/dstackai/dstack/master/.skills/dstack/SKILL.md -o ~/.claude/skills/dstack/SKILL.md
```

## Project structure

```
program.md              — agent instructions (the optimization loop)
dstack-baseline.yml     — baseline dstack service configuration
sessions/
  .gitkeep
  debug-<ISO>.md        — per-session findings and planned changes (created by agent)
  dstack-<ISO>.yml      — per-session optimized configurations (created by agent)
```

## Running the agent

Point your AI agent at this repo and prompt:

```
Read program.md and let's set up an optimization session.
```

The agent will verify prerequisites, deploy the initial service, and begin the autonomous optimization loop. You can walk away — it will keep iterating, logging results, and redeploying improvements until you stop it.

## License

MIT
