# Batch Processing

llm-d supports offline and batch inference workloads through two components that can be deployed independently or together:

- **[Batch Gateway](batch-gateway.md)** — an OpenAI-compatible Batch API (`/v1/batches`, `/v1/files`) for submitting, tracking, and managing batch inference jobs. Includes a processor that dispatches individual inference requests to the Inference Gateway.

- **[Async Processor](async-processor.md)** — a lightweight dispatch agent that pulls individual inference requests from queues and sends them to the Inference Gateway based on system metrics, enabling flow-control-aware async processing.

When deployed together, the Batch Gateway hands off individual requests to the Async Processor instead of dispatching directly, gaining metric-aware flow control.

## Related

- [Batch Gateway Deployment Guide](../../../../../guides/batch-gateway/README.md)
- [Async Processor Deployment Guide](../../../../../guides/asynchronous-processing/README.md)
