# eks-llm-inference

> Run large language models on EKS with vLLM: GPU node provisioning, model serving, autoscaling, and cost controls.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

## What this covers

- **GPU NodePool** — Karpenter provisions `g5`/`g6` Spot nodes on demand
- **vLLM deployment** — serves any HuggingFace model with OpenAI-compatible API
- **KEDA autoscaling** — scale replicas based on pending request queue depth
- **Model caching** — S3-backed model cache via `s5cmd` init container (avoids re-downloading on node recycle)
- **Cost controls** — Spot-first, scale to zero overnight via KEDA ScaledObject min=0

## Architecture

```
Client Request (OpenAI API format)
        │
        ▼
┌───────────────────────┐
│  Kubernetes Service   │
│  (ClusterIP / NLB)    │
└──────────┬────────────┘
           │
           ▼
┌───────────────────────────────────────────────────┐
│  vLLM Pod                                         │
│  ┌──────────────────┐  ┌────────────────────────┐ │
│  │  init: s5cmd     │  │  vllm serve            │ │
│  │  (pull model     │  │  --model /mnt/model    │ │
│  │   from S3 cache) │  │  --tensor-parallel 1   │ │
│  └──────────────────┘  │  --port 8000           │ │
│                        └────────────────────────┘ │
│  Node: g5.xlarge (1x A10G GPU, 24GB VRAM)        │
└───────────────────────────────────────────────────┘
           │
           ▼ KEDA scales based on queue depth
┌───────────────────────┐
│  Karpenter NodePool   │
│  g5/g6 Spot           │
└───────────────────────┘
```

## Quick start

```bash
# 1. Apply GPU NodePool
kubectl apply -f manifests/gpu-nodepool.yaml

# 2. Deploy vLLM (default: Llama-3.1-8B-Instruct)
kubectl apply -f manifests/vllm-deployment.yaml

# 3. Apply KEDA autoscaler
kubectl apply -f manifests/keda-scaledobject.yaml

# 4. Test
kubectl port-forward svc/vllm-service 8000:8000 &
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"meta-llama/Llama-3.1-8B-Instruct","messages":[{"role":"user","content":"Hello!"}]}'
```

## GPU instance guide

| Instance | GPU | VRAM | Best for |
|----------|-----|------|----------|
| g5.xlarge | 1x A10G | 24GB | 7B–13B models |
| g5.12xlarge | 4x A10G | 96GB | 70B models (tensor parallel) |
| g6.xlarge | 1x L4 | 24GB | 7B–13B, better perf/$ than g5 |
| p4d.24xlarge | 8x A100 | 320GB | 405B+ models |

## Files

```
eks-llm-inference/
├── manifests/
│   ├── gpu-nodepool.yaml         # Karpenter GPU NodePool
│   ├── vllm-deployment.yaml      # vLLM Deployment + Service
│   └── keda-scaledobject.yaml    # KEDA autoscaler
└── terraform/
    └── main.tf                   # KEDA, NVIDIA device plugin, S3 model cache bucket
```

## License

MIT — by [Goutham Annem](https://github.com/Goutham-Annem)
