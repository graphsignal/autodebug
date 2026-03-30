# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

**autodebug** is a telemetry-driven inference optimization loop. An AI agent deploys ML inference services via [dstack](https://dstack.ai/), fetches profiling telemetry from [Graphsignal](https://graphsignal.com/), analyzes performance, and redeploys with improved configurations — indefinitely and autonomously.

The agent's operating instructions are in `program.md`. Read that file first.

## Prerequisites

```bash
uv tool install graphsignal-debug && graphsignal-debug login
uv tool install dstack && dstack login
```

Download agent skill files (also read each one at session start):
```bash
mkdir -p ~/.claude/skills/graphsignal-sdk ~/.claude/skills/graphsignal-debug ~/.claude/skills/dstack
curl -sL https://raw.githubusercontent.com/graphsignal/graphsignal-python/main/SKILL.md -o ~/.claude/skills/graphsignal-sdk/SKILL.md
curl -sL https://raw.githubusercontent.com/graphsignal/graphsignal-debug/main/SKILL.md -o ~/.claude/skills/graphsignal-debug/SKILL.md
curl -sL https://raw.githubusercontent.com/dstackai/dstack/master/skills/dstack/SKILL.md -o ~/.claude/skills/dstack/SKILL.md
```

## Key commands

Preview a dstack deployment plan (no changes):
```bash
echo "n" | dstack apply -f <config.yml>
```

Deploy (use `-d` in agent/automated contexts — without it, `dstack apply` blocks indefinitely):
```bash
dstack apply -f dstack-baseline.yml -y -d
```

Check service status and logs:
```bash
dstack ps -v
dstack logs <service-name>
```

Get service endpoint and auth token:
```bash
DSTACK_TOKEN=$(dstack config show --project <project> --json 2>/dev/null | jq -r '.token // empty' || echo "")
SERVICE_URL=$(dstack run get <service-name> --json | jq -r '.service.url')
```

Fetch Graphsignal telemetry for a time window:
```bash
graphsignal-debug fetch --start <ISO> --end <ISO>
```

Benchmark an endpoint:
```bash
curl -s -w "\n%{time_starttransfer} %{time_total}" \
  -X POST "${SERVICE_URL}v1/chat/completions" \
  -H "Authorization: Bearer $DSTACK_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"model":"<MODEL>","messages":[{"role":"user","content":"<PROMPT>"}],"stream":true}'
```

## Architecture

The project has no source code — it is purely configuration and agent orchestration:

- `program.md` — agent instructions defining the full optimization loop (read this)
- `dstack-baseline.yml` — initial service config: SGLang on H100, Qwen/Qwen1.5-0.5B-Chat, port 8000, Graphsignal telemetry via `graphsignal-run` wrapper
- `sessions/debug-<ISO>.md` — created by agent per iteration; metrics, findings, and planned changes
- `sessions/dstack-<ISO>.yml` — created by agent per iteration; complete deployable configs with applied optimizations

**Optimization loop** (from `program.md`): benchmark → check logs → fetch telemetry → analyze → identify improvements → write `sessions/debug-<ISO>.md` → write `sessions/dstack-<ISO>.yml` → deploy → repeat forever.

**NEVER STOP**: Once the optimization loop has begun, do NOT pause to ask the human if you should continue. The loop runs until the human interrupts you, period.

## Session log format

Each `sessions/debug-<ISO>.md` must include an **Analysis** section citing exact data from Graphsignal — specific profiled functions (e.g. `sglang.scheduler.Scheduler.run_batch`), span counters (e.g. `sglang.latency.time_to_first_token`), span attributes (e.g. `sglang.startup.*`), and Prometheus metrics (e.g. `sglang.cache_hit_rate`) with their values, so reasoning is directly traceable to raw telemetry.

## Inference metrics

```
prefill_throughput = input_tokens / TTFT
decode_throughput  = output_tokens / (TTLT - TTFT)
token_throughput   = (input_tokens + output_tokens) / TTLT
```

- High TTFT / low prefill throughput → prefill bottleneck
- High TBOT / low decode throughput → decode bottleneck

Optimization targets: minimize TTFT, TBOT, TTLT; eliminate errors.
