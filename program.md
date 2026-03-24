# autodebug

Telemetry-driven inference troubleshooting and optimization loop. Deploy a model as an inference service via dstack, continuously fetch profiling telemetry from Graphsignal, analyze performance, and redeploy with improved configuration.

## Setup

To set up a new optimization session, work with the user to:

1. **Initialize skills**: Download and read the following SKILL.md files for full context:
   ```bash
   mkdir -p ~/.claude/skills/graphsignal-sdk ~/.claude/skills/graphsignal-debug ~/.claude/skills/dstack
   curl -sL https://raw.githubusercontent.com/graphsignal/graphsignal-python/main/SKILL.md -o ~/.claude/skills/graphsignal-sdk/SKILL.md
   curl -sL https://raw.githubusercontent.com/graphsignal/graphsignal-debug/main/SKILL.md -o ~/.claude/skills/graphsignal-debug/SKILL.md
   curl -sL https://raw.githubusercontent.com/dstackai/dstack/master/skills/dstack/SKILL.md -o ~/.claude/skills/dstack/SKILL.md
   ```
   Read each SKILL.md to understand the tools:
   - `graphsignal-sdk` — Graphsignal SDK for inference observability (profiling, tracing, metrics).
   - `graphsignal-debug` — CLI to fetch debug context (profiles, errors, traces) for a time range.
   - `dstack` — Cloud infrastructure for deploying inference services.

2. **Verify prerequisites**: Confirm the user has logged in to both services:
   ```bash
   graphsignal-debug login
   dstack login
   ```
   If either is not configured, instruct the user to run the respective login command before proceeding.

3. **Deploy initial service**: Apply the initial dstack configuration to deploy the inference service:
   ```bash
   dstack apply -f dstack-baseline.yml
   ```
   Wait for the service to become healthy. Check `dstack ps` and `dstack logs` to confirm the endpoint is live.

4. **Initialize sessions directory**: Ensure the `sessions/` directory exists (it contains `.gitkeep`). Session logs and dstack configs will be written here.

5. **Confirm and go**: Confirm setup looks good, then kick off the optimization loop.

## Optimization loop

LOOP FOREVER:

1. **Benchmark the endpoint**: Get the service endpoint and auth token from dstack:
   ```bash
   dstack run get <service-name> --json
   ```
   Extract `service.url` (the endpoint base URL) and use the dstack token for authentication. The model name comes from the `model` field in the dstack configuration.

   Decide benchmark configuration based on the hypothesis being tested:
   - Parallel requests (e.g. `xargs -P`) for batch size and concurrency tuning.
   - Repeated sequential requests for prefix cache parameter evaluation.
   - Long prompts for prefill time analysis.
   - Short prompts with long completions for decode throughput.
   - Mixed workloads for overall throughput assessment.

   Use the OpenAI-compatible chat completions API with `stream: true` to capture time-to-first-token and token timing:
   ```bash
   DSTACK_TOKEN=$(dstack config show --project <project> --json 2>/dev/null | jq -r '.token // empty' || echo "")
   SERVICE_URL=$(dstack run get <service-name> --json | jq -r '.service.url')

   curl -s -w "\n%{time_starttransfer} %{time_total}" \
     -X POST "${SERVICE_URL}v1/chat/completions" \
     -H "Authorization: Bearer $DSTACK_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"model":"<MODEL>","messages":[{"role":"user","content":"<PROMPT>"}],"stream":true}'
   ```

2. **Check dstack logs**: Run `dstack logs <service-name>` to verify the inference engine is running without errors. Look for OOM events, CUDA errors, model loading failures, or configuration warnings.

3. **Fetch telemetry**: Use `graphsignal-debug` to fetch performance data for the benchmark window:
   ```bash
   graphsignal-debug fetch --start <START_ISO> --end <END_ISO>
   ```
   The output includes profiling data, traces, metrics, and errors. Use this as the primary source for performance analysis.

4. **Analyze performance**: Using the telemetry data and benchmark results, analyze with the goal to:
   - Minimize time to first token (TTFT) — fast initial response.
   - Minimize time between output tokens (TBOT) — smooth streaming.
   - Minimize time to last token (TTLT) — fast total completion.
   - Maximize throughput (prefill, decode, and token throughput).
   - Eliminate errors.

   Compute inference metrics (see [Computing inference metrics](#computing-inference-metrics)) and compare against previous sessions.

5. **Identify improvements**: Based on the analysis, identify changes to:
   - Inference engine configuration: batch size, max sequences, prefix caching, chunked prefill, speculative decoding, quantization, attention backend, scheduling policy.
   - dstack configuration: GPU type/count, tensor parallelism, memory allocation, image selection.
   - Request parameters: max tokens, temperature (if affecting engine paths).

6. **Log findings**: Create a new session log file `sessions/debug-<ISO>.md` for the current iteration:
   ```markdown
   # debug-2026-03-22T15:00:00Z

   **Metrics**: TTFT=0.45s, TBOT=0.032s, TTLT=2.1s, Errors=0

   **Analysis** (cite exact data from Graphsignal debug context):
   - Profile `sglang.scheduler.Scheduler.run_batch`: avg=3.47ms/step → scheduler overhead per decode step
   - Profile `sglang.e2e.openai_v1_chat_completions`: avg=384µs → endpoint handling is negligible
   - Span counter `sglang.latency.time_to_first_token`: p95=420ms → prefill bottleneck
   - Span counter `sglang.latency.time_in_model_prefill`: avg=85ms with `sglang.usage.prompt_tokens`=950
   - Span attribute `sglang.startup.enable_mixed_chunk`: false → prefill/decode not overlapping
   - Prometheus metric `sglang.cache_hit_rate`: 0.85 → prefix cache effective but TTFT still high

   **Findings**: High TTFT suggests prefill bottleneck. KV cache utilization at 85%.

   **Planned changes**: Enable chunked prefill, increase max-prefill-tokens to 4096.
   ```
   Always include an **Analysis** section that cites exact data from the Graphsignal debug context. Reference the specific profiled functions (e.g. `sglang.scheduler.Scheduler.run_batch`), span counters (e.g. `sglang.latency.time_to_first_token`, `sglang.usage.prompt_tokens`), span attributes (e.g. `sglang.startup.*` options), and Prometheus metrics (e.g. `sglang.*`) with their values, so the reasoning is directly traceable to the raw telemetry.

7. **Write optimized configuration**: Create a new dstack configuration file in the sessions directory reflecting the improvements:
   ```bash
   # filename: sessions/dstack-2026-03-22T15:00:00Z.yml
   ```
   The file should be a complete, deployable dstack service configuration with the identified changes applied.

8. **Deploy changes**: Apply the optimized configuration:
   ```bash
   dstack apply -f sessions/dstack-<ISO-date>.yml
   ```
   Wait for the service to become healthy before starting the next iteration.

9. **Repeat**: Go back to step 1. If you run out of ideas, revisit previous session logs in `sessions/debug-*.md`, try combining near-miss improvements, explore more radical configuration changes, or vary the benchmark workload to uncover new bottlenecks.

**NEVER STOP**: Once the optimization loop has begun, do NOT pause to ask the human if you should continue. The human might be away and expects you to continue working *indefinitely* until manually stopped. You are autonomous. The loop runs until the human interrupts you, period.

## Computing inference metrics

Extract these inference metrics from the debug context fetched from Graphsignal. Token counts come from the request/response; latencies come from profiling data and benchmark timing.

TTLT may also be available as e2e latency or duration in the Graphsignal debug context.

