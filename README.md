# MULTI-PROVIDER BENCHMARK: LATENCY & TOKEN OPTIMIZATION IN MULTI-AGENT SYSTEMS
# Sequential Baseline vs. Optimized DAG (LangGraph + Prompt Caching + MCP Filter)
# Providers: Anthropic · OpenAI  (Google/Gemini arm implemented but not run in this study)
# Target: Google Colab (Python 3.10+)

## WHAT THIS DOES
Runs the IDENTICAL "Sequential Baseline vs Optimized DAG" workflow across a 4-model
matrix, holding the model constant within each run so the measured delta reflects the
ARCHITECTURE (parallel DAG + prompt caching + MCP payload filtering), not model tiering.

    15 prompts × 4 models × 2 systems = 120 rows → multi_provider_benchmark.csv
    4 repeated runs → 480 measured rows total (used for median-based robust estimation)

Each baseline row = 3 LLM calls; each optimized row = 4 LLM calls (3 workers + reducer).
A full live run issues 15 × 4 × (3 + 4) = 420 API calls per run. PLAN COST ACCORDINGLY.
Use DRY_RUN=True to validate the full pipeline and CSV shape for $0 first.

## PER-PROVIDER FACTS (verified against current docs)

TOKEN METADATA — LangChain normalizes providers into the standardized
AIMessage.usage_metadata: input_tokens / output_tokens / total_tokens, plus
input_token_details.{cache_read, cache_creation}. In this field, input_tokens
INCLUDES cached tokens (unlike the raw Anthropic SDK, which excludes them).
Provider-specific raw parse of response_metadata is used as a documented fallback:
  - OpenAI    : response_metadata['token_usage'] → prompt_tokens / completion_tokens;
                cached at token_usage['prompt_tokens_details']['cached_tokens']
  - Anthropic : response_metadata['usage'] → input_tokens / output_tokens;
                cache_read_input_tokens / cache_creation_input_tokens
                (raw Anthropic input_tokens EXCLUDES cache → normalized to inclusive)

CACHING MECHANICS — Anthropic requires an explicit flag; OpenAI is automatic:
  - Anthropic : explicit cache_control={"type":"ephemeral"} on the system block.
                Min cacheable prefix: claude-haiku-4-5 = 4096 tok, claude-sonnet-4-6 = 1024 tok.
  - OpenAI    : AUTOMATIC for prompts ≥1024 tok. No flag exists or is needed.
                We rely on automatic prefix caching (longest previously computed prefix reused).

The shared system block is sized at ~4,621 tokens, clearing every provider's minimum
so caching engages on all models in the optimized arm.

LANGGRAPH FAN-IN — parallel workers return DELTAS ONLY; the accumulated channels
(worker_outputs, all_turn_metrics) use Annotated[..., operator.add] reducers,
so the parallel fan-in never raises InvalidUpdateError.

NOTE: Google/Gemini arm is implemented in code but was not executed in this study.
All reported results cover Anthropic (claude-haiku-4-5, claude-sonnet-4-6) and
OpenAI (gpt-4o-mini, gpt-4o) only. The Google rows remain commented out for
future reproducibility.
