# Batch Gateway

Process large-scale batch inference jobs via an OpenAI-compatible API — decoupled from real-time serving, with intelligent flow control that fills idle capacity without impacting interactive workloads.

This path is for operators who want to **adopt** batch inference processing in an existing llm-d deployment. For the deployment guide and Helm chart details, see the [guides/batch-gateway](../../../../guides/batch-gateway/README.md). For the architecture and internals, see [architecture/advanced/batch](../../architecture/advanced/batch/batch-gateway.md).

## When to Pick This Path

Pick it when:

- You have **large-scale offline inference workloads** (evaluations, embeddings, dataset processing) that don't need real-time responses.
- You want to **fill idle accelerator capacity** with batch work while protecting interactive traffic from interference.
- Your clients expect an **OpenAI-compatible Batch API** (`/v1/batches`, `/v1/files`) for job submission and tracking.
- You need **multi-tenant isolation** — each tenant's jobs, files, and results are separated.

Skip it when:

- Your workload needs **real-time or low-latency responses** — use the inference gateway directly.
- You need **individual async request/response** without the concept of batch jobs — consider [Asynchronous Processing](../../../../guides/asynchronous-processing/README.md) instead.

## Prerequisites

- A working Inference Gateway, `InferencePool`, and at least one model server. If you don't have this, start with [getting-started/quickstart.md](../../getting-started/quickstart.md).
- PostgreSQL (12+) and Redis (6+) accessible from the cluster.
- S3-compatible storage or a shared PVC for batch input/output files.
- Helm 3.0+.

## Deploy

Install via Helm, pointing the processor at your inference gateway:

```bash
export NAMESPACE=batch-gateway
export INFERENCE_GW_URL="http://infra-inference-scheduling-inference-gateway-istio.llm-d-inference-scheduler.svc.cluster.local:80"

kubectl create namespace ${NAMESPACE}

# Create the required secret
kubectl create secret generic batch-gateway-secrets -n ${NAMESPACE} \
  --from-literal=redis-url="redis://redis-master:6379/0" \
  --from-literal=postgresql-url="postgresql://user:pass@postgresql:5432/batchgateway"

# Install
helm install batch-gateway oci://ghcr.io/llm-d-incubation/charts/batch-gateway \
  -n ${NAMESPACE} \
  --set processor.config.globalInferenceGateway.url="${INFERENCE_GW_URL}" \
  --set "apiserver.config.batchAPI.passThroughHeaders={Authorization}"
```

### Authentication Passthrough

The processor forwards headers listed in `passThroughHeaders` with every inference request. This ensures the downstream gateway can authenticate and authorize the end user. Include all authentication headers your gateway expects (e.g., `Authorization`, `X-MaaS-Subscription`).

### Per-Model Routing

If different models are served by different gateways, use `modelGateways` instead of `globalInferenceGateway`:

```yaml
processor:
  config:
    modelGateways:
      "llama-3":
        url: "https://gateway-a.default.svc.cluster.local:8443"
        requestTimeout: "5m"
        maxRetries: 3
      "mistral-7b":
        url: "https://gateway-b.default.svc.cluster.local:8443"
        requestTimeout: "5m"
        maxRetries: 3
```

### Scaling the Processor

Adjust concurrency based on your cluster capacity:

```yaml
processor:
  replicaCount: 3
  config:
    numWorkers: 50
    globalConcurrency: 500
    perModelMaxConcurrency: 20
```

## Send Requests

Upload an input file and create a batch job:

```bash
# Upload input file (JSONL format)
FILE_ID=$(curl -s -X POST http://<apiserver>/v1/files \
  -F "purpose=batch" \
  -F "file=@batch_input.jsonl" | jq -r '.id')

# Create batch job
BATCH_ID=$(curl -s -X POST http://<apiserver>/v1/batches \
  -H "Content-Type: application/json" \
  -d "{
    \"input_file_id\": \"${FILE_ID}\",
    \"endpoint\": \"/v1/chat/completions\",
    \"completion_window\": \"24h\"
  }" | jq -r '.id')

# Check status
curl -s http://<apiserver>/v1/batches/${BATCH_ID} | jq '{status, request_counts}'
```

The expected job status flow is: `validating` → `in_progress` → `finalizing` → `completed`.

## Verify

Once jobs are flowing, confirm three things in Prometheus:

1. **Jobs are being processed.** `jobs_processed_total{result="success"}` is incrementing. If it stays at zero, check the processor logs for queue connectivity or inference gateway errors.
2. **Queue wait time is acceptable.** `job_queue_wait_duration_seconds` should be within your SLO. High values indicate the processor can't keep up — increase `replicaCount` or `globalConcurrency`.
3. **Inference requests are succeeding.** `model_request_execution_duration_seconds{model}` shows healthy latency distribution. Check `request_errors_by_model_total{model}` for failures.

Pre-built Grafana dashboards are included in the Helm chart — enable with `grafana.dashboards.enabled=true`.

## Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| Pods running but no jobs processed | Processor can't reach inference gateway — check `globalInferenceGateway.url` and network policies. |
| Jobs stuck in `validating` | API server can't connect to PostgreSQL or Redis — check the `batch-gateway-secrets` Secret and database connectivity. |
| Jobs complete but output file empty | Inference requests failing — check processor logs for HTTP errors from the inference gateway. Verify `passThroughHeaders` includes the required auth headers. |
| High queue wait time | Processor under-provisioned — increase `replicaCount`, `numWorkers`, or `globalConcurrency`. |
| `403` on inference requests | Authorization failing at the inference gateway — ensure the batch route's auth headers are forwarded and the user has access to the requested model. |
| GC not cleaning up expired jobs | Check GC pod logs. Verify `gc.enabled=true` and the GC pod is running. |

## Related

- [Batch Gateway Architecture](../../architecture/advanced/batch/batch-gateway.md) — components, data flow, and processing pipeline.
- [Batch Gateway Repository](https://github.com/llm-d-incubation/batch-gateway) — source code, Helm chart, platform-specific deployment guides, and demo scripts.
- [Asynchronous Processing](../../../../guides/asynchronous-processing/README.md) — complementary queue-based async inference for individual requests.
