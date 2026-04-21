# Experimental Feature: Batch Gateway

[Batch Gateway](https://github.com/llm-d-incubation/batch-gateway) provides an OpenAI-compatible Batch API for submitting, tracking, and managing large-scale batch inference jobs. It is designed to process batch workloads alongside interactive traffic, minimizing interference with real-time requests while meeting batch jobs' SLOs.

## Overview

Batch Gateway integrates with llm-d to:

- **Provide an OpenAI-compatible API**: Full schema parity with OpenAI's `/v1/batches` and `/v1/files` endpoints.
- **Process large-scale batch jobs**: Support for up to 50,000 requests per job, with progress tracking and job management.
- **Optimize resource utilization**: Intelligent flow control monitors downstream metrics and adjusts batch dispatch volume to fill idle capacity without impacting interactive workloads.
- **Multi-tenant isolation**: Per-tenant data isolation with pluggable authentication and authorization.

### Components

Batch Gateway consists of three independently scalable components:

1. **API Server** (`batch-gateway-apiserver`) — REST API for batch job submission, management, tracking, and file management.
2. **Batch Processor** (`batch-gateway-processor`) — Pulls jobs from a priority queue, builds per-model execution plans, dispatches inference requests to the inference gateway, and writes results.
3. **Garbage Collector** (`batch-gateway-gc`) — Cleans up expired jobs and files.

### Data Layer

Batch Gateway uses pluggable storage backends. Each function is backed by a single plug-in, chosen at deployment time:

| Function | Available plug-ins |
|----------|-------------------|
| Jobs and files metadata | PostgreSQL, Redis |
| Priority queue, events, status updates | Redis |
| File storage (input/output) | S3, Filesystem |

## Prerequisites

Before installing Batch Gateway, ensure you have:

1. **Kubernetes cluster**: A running Kubernetes cluster (v1.25+).
   - For local development, you can use **Kind** or **Minikube**.
   - For production, GKE, AKS, or OpenShift are supported.
2. **Helm**: 3.0+
3. **llm-d Inference Stack**: Batch Gateway requires an existing [Intelligent Inference Scheduling](../inference-scheduling/README.md) stack to dispatch requests to.
4. **PostgreSQL**: 12+ for metadata storage (or Redis as an alternative).
5. **Redis**: 6+ for priority queue, events, and status updates.
6. **S3 or Filesystem**: For batch input and output file storage.

## Installation

### Step 1: Create the Namespace

```bash
export NAMESPACE=batch-gateway
kubectl create namespace ${NAMESPACE}
```

### Step 2: Create the Secrets

Batch Gateway requires a Kubernetes Secret with database and storage credentials:

```bash
kubectl create secret generic batch-gateway-secrets -n ${NAMESPACE} \
  --from-literal=redis-url="redis://redis-master.redis.svc.cluster.local:6379/0" \
  --from-literal=postgresql-url="postgresql://user:password@postgresql.postgresql.svc.cluster.local:5432/batchgateway" \
  --from-literal=s3-secret-access-key="<your-s3-secret-key>"
```

### Step 3: Configure the Inference Gateway URL

The Batch Processor needs to know where to send inference requests. This is configured via the `processor.config.globalInferenceGateway.url` Helm value, or per-model via `processor.config.modelGateways`.

**Single gateway** (all models route to one endpoint):

```bash
export INFERENCE_GW_URL="http://infra-inference-scheduling-inference-gateway-istio.llm-d-inference-scheduler.svc.cluster.local:80"
```

**Per-model gateways** (different endpoints per model): see the [Helm chart README](https://github.com/llm-d-incubation/batch-gateway/blob/main/charts/batch-gateway/README.md) for `modelGateways` configuration.

### Step 4: Deploy

**From OCI Registry:**

```bash
helm install batch-gateway oci://ghcr.io/llm-d-incubation/charts/batch-gateway \
  -n ${NAMESPACE} \
  --set processor.config.globalInferenceGateway.url="${INFERENCE_GW_URL}" \
  --set "apiserver.config.batchAPI.passThroughHeaders={Authorization}" \
  --set global.fileClient.fs.pvcName="batch-gateway-pvc"
```

**From Source:**

```bash
helm install batch-gateway ./charts/batch-gateway \
  -n ${NAMESPACE} \
  --set processor.config.globalInferenceGateway.url="${INFERENCE_GW_URL}" \
  --set "apiserver.config.batchAPI.passThroughHeaders={Authorization}" \
  --set global.fileClient.fs.pvcName="batch-gateway-pvc"
```

> **Note**: `passThroughHeaders` should include any authentication headers (e.g., `Authorization`) that the inference gateway expects. The processor forwards these headers when dispatching individual inference requests.

## Verification

1. Check that all pods are running:

   ```bash
   kubectl get pods -n ${NAMESPACE}
   ```

   You should see pods for `apiserver`, `processor`, and `gc`, all in `Running` state.

2. Verify health endpoints:

   ```bash
   kubectl port-forward -n ${NAMESPACE} svc/batch-gateway-apiserver 8081:8081 &
   curl http://localhost:8081/health
   ```

## Usage

### Upload an Input File

Prepare a JSONL input file with one request per line (see the [OpenAI Batch API format](https://platform.openai.com/docs/api-reference/batch)):

```json
{"custom_id": "req-001", "method": "POST", "url": "/v1/chat/completions", "body": {"model": "my-model", "messages": [{"role": "user", "content": "Hello"}], "max_tokens": 100}}
{"custom_id": "req-002", "method": "POST", "url": "/v1/chat/completions", "body": {"model": "my-model", "messages": [{"role": "user", "content": "What is llm-d?"}], "max_tokens": 200}}
```

Upload the file:

```bash
curl -X POST http://localhost:8000/v1/files \
  -F "purpose=batch" \
  -F "file=@batch_input.jsonl"
```

### Create a Batch Job

```bash
curl -X POST http://localhost:8000/v1/batches \
  -H "Content-Type: application/json" \
  -d '{
    "input_file_id": "<file-id-from-upload>",
    "endpoint": "/v1/chat/completions",
    "completion_window": "24h"
  }'
```

### Monitor Job Status

```bash
curl http://localhost:8000/v1/batches/<batch-id> | jq '{status, request_counts}'
```

### Download Results

Once the job status is `completed`, retrieve the output file:

```bash
# Get the output file ID
OUTPUT_FILE_ID=$(curl -s \
  http://localhost:8000/v1/batches/<batch-id> | jq -r '.output_file_id')

# Download the results
curl http://localhost:8000/v1/files/${OUTPUT_FILE_ID}/content > results.jsonl
```

## Cleanup

```bash
helm uninstall batch-gateway -n ${NAMESPACE}
kubectl delete namespace ${NAMESPACE}
```

## Platform-Specific Deployment Guides

For production deployments with authentication, authorization, and TLS, see the detailed guides in the [batch-gateway repository](https://github.com/llm-d-incubation/batch-gateway/tree/main/docs/guides):

- [Vanilla Kubernetes](https://github.com/llm-d-incubation/batch-gateway/blob/main/docs/guides/deploy-k8s.md) — Istio + Kuadrant + cert-manager
- [Red Hat OpenShift AI (RHOAI)](https://github.com/llm-d-incubation/batch-gateway/blob/main/docs/guides/deploy-rhoai.md) — OpenShift + RHOAI operator
- [Models-as-a-Service (MaaS)](https://github.com/llm-d-incubation/batch-gateway/blob/main/docs/guides/deploy-maas.md) — MaaS platform with API key auth

## Related

- [Batch Gateway Repository](https://github.com/llm-d-incubation/batch-gateway) — source code, Helm chart, and detailed documentation.
- [Asynchronous Processing](../asynchronous-processing/README.md) — queue-based async inference for individual requests (complementary to Batch Gateway's job-oriented API).
