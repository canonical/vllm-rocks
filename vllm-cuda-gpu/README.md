## vllm-cuda-gpu

Canonical rock for [vLLM](https://github.com/vllm-project/vllm) v0.32.0, GPU/CUDA edition.
It packages the prebuilt vLLM CUDA wheel as an OpenAI-compatible inference server and
targets NVIDIA GPUs on `linux/amd64` (x86_64) with CUDA 13.0 on Ubuntu 24.04.
The host NVIDIA driver must be compatible with CUDA 13.

For a CPU-only build, see [`../vllm-cpu`](../vllm-cpu).

### Feature scope

This rock provides vLLM 0.32.0 with CUDA 13.0 and NIXL 1.5.0. It is designed for
ordinary tensor-parallel and llm-d disaggregated-prefill/decode deployments.
The following upstream features are not included:

* **LMCache and Mooncake:** NIXL is the tested KV connector. These are alternative
  or optional connectors.
* **DeepEP / expert-parallel kernels:** Not required for ordinary tensor parallelism
  or the tested multi-GPU deployment. Add a separate NVSHMEM/DeepEP build before
  using wide expert-parallel or DeepEP workloads.
* **GDRCopy:** Not required for basic GPU tensor parallelism or the tested llm-d
  deployment. It provides GPU-direct transfer acceleration for suitable RDMA/NIXL
  deployments.
* **CRIU snapshot runtime:** This rock does not implement vLLM checkpoint/snapshot
  workflows. It was not included in the existing v0.19.0 rock either.
* **`UV_COMPILE_BYTECODE=1`:** Omitted as a cold-start optimization. The Python
  environment is root-owned, so bytecode caching is not required for correctness.
* **`TORCH_CUDA_ARCH_LIST`:** Omitted because the CUDA 13.0/B200 deployment works
  without a fixed multi-architecture JIT target.
* **Upstream examples, benchmarks, and `collect_env.py`:** Convenience assets only;
  they are not required by the KServe/llm-d serving path.
* **`/usr/local/bin/vllm-rs`:** The charmed deployment uses the standard vLLM Python
  entrypoint. This alias was not present in the v0.19.0 rock.
* **Published-image NVIDIA compatibility, linker, and runtime uv settings:** These
  add parity and robustness but are not needed for the tested host and deployment.
  Runtime uv settings can be added if package installation inside the image becomes
  a supported use case.

### Testing

#### Prerequisites

* an NVIDIA GPU with a compatible driver
* the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
  (so that `docker run --gpus all` exposes the GPU to the container)
* docker

#### Instructions

Launch the server with a small model (downloads from the HuggingFace Hub on first run):

```bash
docker run --gpus all -p 8000:8000 \
  -e HF_HOME=/tmp/huggingface \
  vllm-cuda-gpu:0.32.0 \
  --model facebook/opt-125m
```

Test with the OpenAI-compatible completions endpoint:

```bash
curl -H "content-type: application/json" \
  localhost:8000/v1/completions \
  -d '{"model": "facebook/opt-125m", "prompt": "The capital of France is", "max_tokens": 20}'
```

### Log forwarding

The rock can forward the vLLM server logs to a [Grafana Loki](https://grafana.com/oss/loki/)
instance using [Pebble log forwarding](https://ubuntu.com/docs/pebble/how-to/forward-logs-to-loki/).
Forwarding is opt-in: set the `LOKI_URL` environment variable to the Loki push
endpoint and the container adds a Pebble `log-targets` layer at startup. When
`LOKI_URL` is unset, forwarding is disabled and the server starts normally.

```bash
docker run --gpus all -p 8000:8000 \
  -e HF_HOME=/tmp/huggingface \
  -e LOKI_URL=http://my-loki:3100/loki/api/v1/push \
  vllm-cuda-gpu:0.32.0 \
  --model facebook/opt-125m
```

Forwarded logs carry Pebble's default `pebble_service` label plus `app=vllm`,
`edition=gpu`, `version=0.32.0`, and `pod=$HOSTNAME`.

### Building

> **Note:** Building this rock downloads CUDA runtime wheels and precompiled
> FlashInfer kernels. The build itself runs on CPU and does not require a GPU;
> only running the server does.

```bash
tox -e pack
tox -e export-to-docker
tox -e sanity
```

`tox -e sanity` runs on any machine (it only checks that the expected files are present
in the rock).
