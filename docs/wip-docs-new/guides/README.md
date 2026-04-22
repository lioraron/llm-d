# Guides

A **guide** is a documented, tested, and benchmarked deployment pattern for large scale LLM serving.

These guides are targeted at production serving that want to achieve SOTA performance while minimizing operational complexity. They help identify key optimizations, understand their tradeoffs, and verify the gains against your own workload.

Each guide is hosted in [llm-d guides](https://github.com/llm-d/llm-d/tree/main/guides).

## Guides

* [**Intelligent Inference Scheduling**](./intelligent-inference-scheduling.md) -- Prefix-cache and load-aware request scheduling.
* [**Flow Control**](./flow-control.md) -- Prioritize traffic from multiple tenants on the same server resources.
* [**P/D Disaggregation**](./pd-disaggregation.md) -- Separate prefill and decode phases of inference into separate instances.
* [**Multi-Node Wide Expert Parallelism**](./wide-expert-parallelism.md) -- Deploy large MoE models over multiple nodes with DP/EP.
* [**KV Cache Management**](./kv-cache-management.md) -- Offload KV caches to CPU RAM and storage for increased cache hit rates.

## Experimental

* [**Predicted Latency-Based Scheduling**](./experimental/predicted-latency.md) -- Use an online-trained XGBoost model for latency-aware scheduling decisions.
* [**Batch Gateway**](./experimental/batch-gateway.md) -- Process large-scale batch inference jobs via an OpenAI-compatible Batch API.

> [!IMPORTANT]
> The deployment guides are intended to be a starting point for your own configuration and deployment of model servers.
