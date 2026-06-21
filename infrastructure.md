# AI Lab — Infrastructure Documentation

> **Version:** 2026-06-20  
> **Author:** Chuck — AI Infrastructure Architect  
> **Owner:** rio  
> **Status:** 🟢 Production  

This document provides a complete reference for the AI lab infrastructure, covering all nodes, network topology, model deployments, deployment recipes, the RAG pipeline, and operational procedures.

---

## Table of Contents

1. [System Inventory](#1-system-inventory)
2. [Network Map](#2-network-map)
3. [Model Deployment Registry](#3-model-deployment-registry)
4. [Architecture Diagram](#4-architecture-diagram)
5. [Deployment Recipes](#5-deployment-recipes)
6. [RAG Pipeline](#6-rag-pipeline)
7. [Monitoring & Maintenance](#7-monitoring--maintenance)

---

## 1. System Inventory

### 1.1 — Spark (Primary GPU Node)

| Attribute | Value |
|---|---|
| **Hostname** | `spark.garage.loc` |
| **IP Address** | `10.20.20.100` (enP7s7) |
| **Docker Bridge** | `172.17.0.1/16` (docker0) |
| **OS** | Ubuntu 24.04.4 LTS (Noble Numbat) |
| **Kernel** | Linux 6.17.0-1021-nvidia (aarch64) |
| **CPU** | ARM — 8 cores (NVIDIA GB10 integrated) |
| **GPU** | NVIDIA GB10 — UUID `GPU-2a5674fb-9167-2650-9bc8-e20a25cd3ca7` |
| **GPU Memory** | 128 GB unified (shared RAM + VRAM) |
| **System RAM** | 121 GB total (9.7 GB available) |
| **Swap** | 15 GB (6.2 GB used) |
| **Driver** | NVIDIA 580.159.03 / CUDA 13.0 |
| **SSH** | `ssh rio@spark.garage.loc` (key auth) |

**Notes:**
- The GB10 uses unified memory — the GPU and CPU share the same physical RAM pool.
- 68.8 GB is currently dedicated to the primary vLLM engine (6.8 TB/s bandwidth).
- GPU currently idling at ~54°C, ~14 W power draw.

---

### 1.2 — Nebo (Proxmox Hypervisor)

| Attribute | Value |
|---|---|
| **Hostname** | `nebo.garage.loc` |
| **IP Address** | `10.20.20.10` |
| **OS** | Debian 13 (Trixie) — Proxmox VE 7.0.2 |
| **CPU** | 64 cores |
| **System RAM** | 250 GB total (170 GB available) |
| **Disk (root)** | 20 GB — 6.0 GB used (33%) |
| **VM Storage** | `datastore1` — 250 GB scsi0 (VM 100) |
| **SSH** | `ssh root@nebo.garage.loc` |
| **Proxmox API** | `https://nebo.garage.loc:8006/api2/json` |
| **Portainer** | `https://10.20.20.11:9443` (VM 100) |

#### Proxmox Virtual Machines

| VMID | Name | Type | Status | RAM | Disk | Cores | Notes |
|---|---|---|---|---|---|---|---|
| 100 | **rock** | QEMU | Running | 64 GB | 250 GB | 8 | Hosts Portainer, data volumes, RAG tools |
| 200 | **rag-vm** | LXC | — | — | — | — | Ubuntu 26.04 — see §6 |

---

### 1.3 — Nuzi (Local Mac — Development / Light Inference)

| Attribute | Value |
|---|---|
| **Hostname** | `nuzi.garage.loc` |
| **IP Address** | `10.20.20.101` |
| **Machine** | Mac Mini M2 Pro (Mac14,12) |
| **Chip** | Apple M2 Pro |
| **Memory** | 16 GB unified |
| **CPU Cores** | 10 (8 performance + 2 efficiency) |
| **Disk** | ~2 GB used on `/` (development / scratch) |
| **SSH** | `ssh rio@nuzi.garage.loc` (key auth) |

**Notes:**
- Primary development machine for writing, testing, and debugging.
- Can run lightweight models locally via llama.cpp or MLX.
- Not used for heavy inference workloads.

---

## 2. Network Map

### 2.1 — Topology

```
                          ┌─────────────────┐
                          │   10.20.20.0/24  │
                          │   (Garage LAN)   │
                          └──┬───┬─────┬─────┘
                             │   │     │
                   ┌─────────┘   │     └──────────┐
                   │             │                │
             ┌─────▼─────┐  ┌───▼────┐    ┌──────▼──────┐
             │  Spark     │  │ Nebo   │    │    Nuzi     │
             │ GB10       │  │ Proxmox│    │ Mac M2 Pro  │
             │ .100       │  │ .10    │    │   .101      │
             └─────┬──────┘  └───┬────┘    └──────┬──────┘
                   │             │                │
            ┌──────┤      ┌──────┤         ┌──────┤
            │      │      │      │         │      │
            ▼      ▼      ▼      ▼         ▼      ▼
      Docker:      │  VM 100   LXC 200  Local dev
      8000→vLLM    │  rock       rag-vm  (llama.cpp)
      8002→vLLM    │  .11         Ubuntu
            │      │  Portainer
            │      │  9443
```

### 2.2 — Host-to-Host Address Book

| Hostname | Alias | IP | Role | Access Method |
|---|---|---|---|---|
| `spark.garage.loc` | spark | 10.20.20.100 | Primary GPU node | SSH (`rio`), HTTP (:8000, :8002) |
| `nebo.garage.loc` | nebo | 10.20.20.10 | Proxmox hypervisor | SSH (`root`) |
| `nuzi.garage.loc` | nuzi | 10.20.20.101 | Mac Mini (dev) | SSH (`rio`), localhost |
| (VM) `rock` | — | 10.20.20.11 | Portainer / RAG host | SSH (inside Proxmox) |
| (LXC) `rag-vm` | — | TBD | ChromaDB / embedding | SSH (inside Proxmox) |
| (Docker) `docker0` | — | 172.17.0.1/16 | Spark Docker bridge | Docker network |

### 2.3 — Port Summary

| Port | Protocol | Service | Host |
|---|---|---|---|
| 22 | SSH | Shell access | spark, nebo, nuzi |
| 8000 | HTTP | vLLM (Qwen3.6-35B NVFP4) | spark |
| 8002 | HTTP | vLLM (Qwen2.5-7B) | spark |
| 8006 | HTTPS | Proxmox API | nebo |
| 9443 | HTTPS | Portainer | rock (VM 100) |

---

## 3. Model Deployment Registry

### 3.1 — Live Models (as of 2026-06-20)

| # | Container | Name (Docker) | Model | Quantization | Port | GPU Mem | Status |
|---|---|---|---|---|---|---|---|
| 1 | `ecstatic_franklin` | spark:8000 | **Qwen3.6-35B-A3B-NVFP4** | NVFP4 (4-bit) | :8000 | ~69 GB | 🟢 Running 9 h |
| 2 | `fast_model` | spark:8002 | **Qwen2.5-7B-Instruct** | Native FP16 | :8002 | ~30 GB | 🟢 Running 5 h |

**Total GPU memory in use:** ~99 GB / 128 GB (~77%)

### 3.2 — Primary Engine Details (Qwen3.6-35B on :8000)

```
Host:          0.0.0.0:8000
Tensor PP:     1 (single GPU)
Max context:   262 144 tokens (~256K)
Max batches:   8 192 tokens
Max sequences: 4

Memory:
  GPU util:    40% (0.4 fraction)
  KV cache:    fp8 dtype (8-bit quantized)

Performance:
  Attention:   flashinfer (GPU-optimized)
  MOE backend: marlin (INT4 quantized MoE kernels)

Caching:
  Prefix:      ✅ enabled (reuses KV for repeated prefixes)
  Prefill:     ✅ chunked (processes long prompts in batches)

Speculative:
  Method:      mtp (Multi-Token Prediction)
  Tokens:      3 speculative tokens
  Backend:     triton (GPU-accelerated)

Loading:
  Format:      fastsafetensors (optimized loading)
  Remote:      ✅ trust-remote-code enabled

Parser:
  Reasoning:   qwen3-parser (native tool-calling)
  Tool call:   qwen3_xml (XML-formatted tool calls)

Other:
  Auto tool:   ✅ enable-auto-tool-choice
  Async:       ✅ enable-async-scheduling
```

### 3.3 — Secondary Engine Details (Qwen2.5-7B on :8002)

```
Host:     0.0.0.0:8002
Max context:   4 096 tokens
Max batches:   8 192 tokens
Max sequences: 16 (higher for bursty lightweight queries)
KV cache:     fp8 dtype
Performance:  prefix caching + chunked prefill enabled
```

### 3.4 — GPU Process Summary

```
GPU UUID: GPU-2a5674fb-9167-2650-9bc8-e20a25cd3ca7
├── VLLM::EngineCore  →  68 794 MiB (~67 GB) — Qwen3.6-35B NVFP4 (port 8000)
├── VLLM::EngineCore  →  29 554 MiB (~29 GB) — Qwen2.5-7B (port 8002)
└── Free:              ~30 652 MiB (~30 GB)  — Available headroom
```

---

## 4. Architecture Diagram

### 4.1 — Full Stack Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CLIENT / ORCHESTRATION                       │
│         (Hermes Agent, LangChain, scripts, APIs, CLI)              │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ HTTP (OpenAI-compatible API)
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         SPARK (GPU Node)                            │
│                    10.20.20.100 :8000                               │
│  ┌─────────────────────────────────────────────────────┐            │
│  │              Docker: vllm/vllm-openai                │            │
│  │                                                     │            │
│  │  ┌─────────────────────────────────────────────┐    │            │
│  │  │  ecstatic_franklin  :8000                   │    │            │
│  │  │  ──────────────────────────────────────     │    │            │
│  │  │  • Qwen3.6-35B-A3B-NVFP4 (4-bit)            │    │            │
│  │  │  • 256K context, chunked prefill            │    │            │
│  │  │  • MTP speculative decoding (×3)            │    │            │
│  │  │  • KV cache: fp8, prefix caching            │    │            │
│  │  └──────────────────┬──────────────────────────┘    │            │
│  │                     │                                │            │
│  │  ┌─────────────────────────────────────────────┐    │            │
│  │  │  fast_model  :8002                          │    │            │
│  │  │  ──────────────────────────────────────     │    │            │
│  │  │  • Qwen2.5-7B-Instruct                      │    │            │
│  │  │  • 4K context, burst-ready                  │    │            │
│  │  └──────────────────┬──────────────────────────┘    │            │
│  │                     │                                │            │
│  │                     ▼                                │            │
│  │          [NVIDIA GB10 — 128 GB unified]               │            │
│  │          GPU mem: ~99 GB used / 30 GB free           │            │
│  └─────────────────────────────────────────────────────┘            │
│  System: 121 GB RAM, Ubuntu 24.04, CUDA 13.0, Driver 580.159        │
└─────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────┐
│  NEBO (Proxmox Hypervisor)  ─  10.20.20.10                          │
│  ┌──────────────────────────┐    ┌──────────────────────┐          │
│  │  VM 100 "rock"           │    │  LXC 200 "rag-vm"    │          │
│  │  Ubuntu / Debian         │    │  Ubuntu 26.04        │          │
│  │  64 GB RAM, 250 GB disk  │    │                      │          │
│  │                          │    │  ┌────────────────┐  │          │
│  │  ┌────────────────────┐  │    │  │ ChromaDB       │  │          │
│  │  │ Portainer (9443)   │  │    │  │ Vector Store   │  │          │
│  │  └────────────────────┘  │    │  └────────────────┘  │          │
│  │  ┌────────────────────┐  │    │  ┌────────────────┐  │          │
│  │  │ RAG Tools /        │  │    │  │ LDS Script-    │  │          │
│  │  │ Data Storage       │  │    │  │ Index /        │  │          │
│  │  └────────────────────┘  │    │  │ Embeddings     │  │          │
│  │                          │    │  └────────────────┘  │          │
│  └──────────────────────────┘    └──────────────────────┘          │
└─────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────┐
│  NUZI (Local Mac)  ─  10.20.20.101                                   │
│  ┌─────────────────────────────────────────────────────┐            │
│  │  Mac Mini M2 Pro — 16 GB RAM                       │            │
│  │                                                     │            │
│  │  • Hermes Agent (orchestration)                     │            │
│  │  • Development / debugging                          │            │
│  │  • Local llama.cpp / MLX inference (lightweight)    │            │
│  │  • Script authoring (chuck.py, prompts, docs)       │            │
│  └─────────────────────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 — Data Flow: RAG Query

```
Client → spark:8000 (HTTP /v1/chat/completions)
    ↓
vLLM Engine (Qwen3.6-35B NVFP4)
    ↓ [generate text + tool calls]
    ↓
Hermes Agent / Orchestrator
    ↓ [RAG pipeline]
    ↓
rock:9443 (Portainer) or directly to rag-vm
    ↓ [vector search]
ChromaDB (rag-vm) ←→ LDS scripture index
    ↓ [retrieved chunks]
    ↓ [injected into prompt context]
    ↓
spark:8000 (next completion call)
    ↓
Final answer → Client
```

---

## 5. Deployment Recipes

### 5.1 — Deploying a New Model on Spark (vLLM Docker)

#### 5.1.1 — Basic Deployment

```bash
# SSH into spark
ssh rio@spark.garage.loc

# 1. Pull the nightly aarch64 image
docker pull vllm/vllm-openai:nightly-aarch64

# 2. Run the model (adjust parameters as needed)
docker run -d --name <container_name> \
  --runtime nvidia \
  --gpus all \
  -p <host_port>:8000 \
  vllm/vllm-openai:nightly-aarch64 \
  <model_identifier> \
  --host 0.0.0.0 \
  --port 8000 \
  --tensor-parallel-size 1 \
  --trust-remote-code \
  --kv-cache-dtype fp8 \
  --attention-backend flashinfer \
  --moe-backend marlin \
  --gpu-memory-utilization 0.4 \
  --max-model-len 8192 \
  --max-num-seqs 8 \
  --max-num-batched-tokens 8192 \
  --enable-chunked-prefill \
  --async-scheduling \
  --enable-prefix-caching
```

#### 5.1.2 — Full-Featured Deployment (Primary Engine Pattern)

```bash
# For large models with tool-calling and MTP:
docker run -d --name primary_model \
  --runtime nvidia \
  --gpus all \
  -p 8000:8000 \
  vllm/vllm-openai:nightly-aarch64 \
  <model_identifier> \
  --host 0.0.0.0 \
  --port 8000 \
  --tensor-parallel-size 1 \
  --trust-remote-code \
  --kv-cache-dtype fp8 \
  --attention-backend flashinfer \
  --moe-backend marlin \
  --gpu-memory-utilization 0.4 \
  --max-model-len 262144 \
  --max-num-seqs 4 \
  --max-num-batched-tokens 8192 \
  --enable-chunked-prefill \
  --async-scheduling \
  --enable-prefix-caching \
  --speculative-config '{"method":"mtp","num_speculative_tokens":3,"moe_backend":"triton"}' \
  --load-format fastsafetensors \
  --reasoning-parser qwen3 \
  --tool-call-parser qwen3_xml \
  --enable-auto-tool-choice
```

#### 5.1.3 — Lightweight Model Deployment

```bash
# Small model for burst/parallel queries:
docker run -d --name light_model \
  --runtime nvidia \
  --gpus all \
  -p 8002:8000 \
  vllm/vllm-openai:nightly-aarch64 \
  <model_identifier> \
  --host 0.0.0.0 \
  --port 8000 \
  --tensor-parallel-size 1 \
  --gpu-memory-utilization 0.25 \
  --max-model-len 4096 \
  --max-num-seqs 16 \
  --enable-prefix-caching \
  --enable-chunked-prefill
```

#### 5.1.4 — Key Parameters Reference

| Parameter | Purpose | Recommended Value |
|---|---|---|
| `--gpu-memory-utilization` | Fraction of GPU mem for model + KV cache | 0.40 (large) / 0.25 (small) |
| `--max-model-len` | Max context window in tokens | 262144 / 4096 / 8192 |
| `--kv-cache-dtype` | KV cache precision | `fp8` (8-bit, saves ~50%) |
| `--attention-backend` | Attention implementation | `flashinfer` (aarch64 optimized) |
| `--moe-backend` | MoE kernel backend | `marlin` (INT4 MoE) or `triton` |
| `--load-format` | Weights loading speed | `fastsafetensors` |
| `--speculative-config` | Speculative decoding | MTP with 3 tokens |
| `--enable-prefix-caching` | Reuse KV for repeated prefixes | Always enable |
| `--enable-chunked-prefill` | Process long prompts in batches | Always enable |

---

### 5.2 — CPU Inference on Nebo (rock VM / rag-vm)

#### 5.2.1 — Using Ollama (Recommended)

```bash
# On rock VM or rag-vm:
# 1. Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# 2. Pull a model
ollama pull qwen2.5:7b-instruct-q4_K_M

# 3. Start serving
ollama serve &

# 4. Query it
curl http://localhost:11434/api/chat -d '{
  "model": "qwen2.5:7b-instruct-q4_K_M",
  "messages": [{"role":"user","content":"Hello!"}],
  "stream": false
}'
```

#### 5.2.2 — Using llama.cpp (Manual)

```bash
# On rock VM or rag-vm:
# 1. Clone and build
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
cmake -B build
cmake --build build --target main -j$(nproc)

# 2. Download a GGUF model
# (from HuggingFace — e.g. qwen2.5-7b-instruct-q4_k_m.gguf)

# 3. Run inference
./build/main \
  -m qwen2.5-7b-instruct-q4_k_m.gguf \
  -p "Write a short story about an AI:" \
  -n 256 \
  -t 8 \
  --ctx-size 4096
```

**Performance note:** CPU inference on nebo will be significantly slower than GPU.
Recommended for: testing, lightweight batch jobs, or as a fallback.

---

### 5.3 — Local Mac Usage (Nuzi)

#### 5.3.1 — Using llama.cpp (Metal)

```bash
# Build with Metal GPU acceleration
cd llama.cpp
cmake -B build -DLLAMA_METAL=ON
cmake --build build --target main -j10

# Run with Metal acceleration (uses Mac GPU)
./build/main \
  -m model.gguf \
  -p "Your prompt here" \
  -n 512 \
  -t 8 \
  --mlock
```

#### 5.3.2 — Using MLX (Apple Silicon Native)

```bash
# Install MLX
pip install mlx mlx-lm

# Run inference (native Apple Silicon, no Metal setup needed)
python3 -c "
from mlx_lm import load, generate
model, tokenizer = load('mlx-community/Qwen2.5-7B-Instruct-4bit')
response = generate(model, tokenizer, prompt='Hello!', max_tokens=256)
print(response)
"
```

#### 5.3.3 — Using Ollama (Simplest)

```bash
brew install ollama
ollama serve &
ollama run qwen2.5:7b-instruct-q4_K_M
```

**Note:** Mac's 16 GB RAM limits local model size. Models up to ~8B parameters (Q4) fit, but larger models will swap.

---

## 6. RAG Pipeline

### 6.1 — Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    LDS Scripture Index                   │
│  ┌──────────────────┐  ┌──────────────────┐             │
│  │ Scripture Text    │  │ Metadata /       │             │
│  │ (Book/Chapter/    │  │ Cross-refs       │             │
│  │  Verse structure) │  │ Bookmarks        │             │
│  └────────┬─────────┘  └────────┬─────────┘             │
│           │                     │                        │
│           ▼                     ▼                        │
│  ┌──────────────────────────────────────┐                │
│  │  Embedding Pipeline                   │                │
│  │  • Text chunking (by verse/chapter)   │                │
│  │  • Embedding via model API            │                │
│  │  • Chunk metadata extraction          │                │
│  └──────────────┬───────────────────────┘                │
│                 │                                         │
└─────────────────┼─────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│  ChromaDB (on rag-vm LXC 200 / rock VM)                 │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Collections:                                    │   │
│  │  • lds_scripture                                 │   │
│  │  • lds_cross_refs                                │   │
│  │  • lds_bookmarks                                 │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### 6.2 — Query Flow

```
1. User sends query → Hermes Agent (nuzi)
        │
2. Agent formats query → spark:8000 /v1/embeddings (or local)
        │
3. Embedding vector → ChromaDB query (rag-vm)
   ┌─── Vector similarity search (cosine / L2)
   │   ┌─── Top-k retrieval (k=4-8 chunks)
   │   │   ┌─── Score threshold filtering
   │   │   └─── Re-ranking (optional)
   ▼   ▼   ▼
4. Retrieved chunks → injected into system prompt context
        │
5. spark:8000 /v1/chat/completions
   ┌─── Full prompt = [system + RAG chunks + user query]
   │   ┌─── vLLM generates response
   ▼   ▼
6. Final answer → User
```

### 6.3 — ChromaDB Connection

```python
# Connect to ChromaDB on rag-vm or rock
import chromadb

# If ChromaDB runs on rag-vm:
client = chromadb.HttpClient(host="10.20.20.xxx", port=8000)

# Or if running locally (rock VM):
client = chromadb.HttpClient(host="localhost", port=8000)

# Get collection
collection = client.get_or_create_collection(name="lds_scripture")

# Query
results = collection.query(
    query_embeddings=[embedding_vector],
    n_results=5,
    where={"book": "LDS_STANDARD_BOOK"},
    include=["documents", "metadatas", "distances"]
)
```

### 6.4 — LDS Scripture Data Structure

Each scripture document is chunked by verse with metadata:

```json
{
  "book": "Book of Mormon",
  "chapter": 1,
  "verse_start": 1,
  "verse_end": 3,
  "text": "..."
}
```

---

## 7. Monitoring & Maintenance

### 7.1 — Health Check Commands

```bash
# === Spark ===

# Full system health check (Chuck agent)
python3 ~/.hermes/scripts/chuck.py check

# GPU memory & utilization
python3 ~/.hermes/scripts/chuck.py gpu

# GPU process memory breakdown
ssh rio@spark.garage.loc \
  "nvidia-smi --query-compute-apps=gpu_uuid,name,used_memory \
   --format=csv"

# GPU temperature & power
ssh rio@spark.garage.loc \
  "nvidia-smi --query-gpu=temperature.gpu,power.draw,power.limit \
   --format=csv,noheader"

# Docker container status
ssh rio@spark.garage.loc "docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'"

# Docker resource usage
ssh rio@spark.garage.loc "docker stats --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}'"

# System resources
ssh rio@spark.garage.loc "free -h && echo '---' && df -h / && echo '---' && uptime"

# vLLM health (if accessible)
curl -s http://spark:8000/v1/models | python3 -m json.tool

# === Nebo (Proxmox) ===

# Proxmox API status check
ssh root@nebo.garage.loc "qm list && echo '---' && pct list"

# VM 100 rock status
ssh root@nebo.garage.loc "qm status 100"

# Proxmox node resources
ssh root@nebo.garage.loc "qm list && pct list && echo '---' && pvesm status"

# Portainer access
open https://10.20.20.11:9443

# === Nuzi (Mac) ===

# Local system status
sysctl hw.memsize hw.ncpu vm.loadavg
df -h /
sysctl hw.model

# Active processes
top -l 1 -n 10 | head -20
```

### 7.2 — GPU Memory Troubleshooting

#### High GPU Utilization

```bash
# SSH into spark and check
ssh rio@spark.garage.loc "nvidia-smi --query-compute-apps=gpu_uuid,name,used_memory --format=csv"

# If OOM or high memory:
# 1. Identify which container is the culprit
ssh rio@spark.garage.loc "docker ps --format '{{.ID}} {{.Names}} {{.Ports}}'"

# 2. Check container GPU utilization
ssh rio@spark.garage.loc "docker stats <container_name>"

# 3. Kill & restart if needed (BACKUP FIRST!)
ssh rio@spark.garage.loc "docker kill <container_name>"
ssh rio@spark.garage.loc "docker rm <container_name>"
# Then redeploy with lower --gpu-memory-utilization
```

#### GPU Memory Leak Detection

```bash
# Check for memory leaks over time
ssh rio@spark.garage.loc "nvidia-smi --query-gpu=memory.used,temperature.gpu,utilization.gpu --format=csv -l 5"
```

### 7.3 — Container Management

```bash
# List all containers
ssh rio@spark.garage.loc "docker ps -a"

# Stop a container gracefully
ssh rio@spark.garage.loc "docker stop fast_model"

# Remove a stopped container
ssh rio@spark.garage.loc "docker rm fast_model"

# Clean up dangling images
ssh rio@spark.garage.loc "docker image prune -f"

# Check container logs
ssh rio@spark.garage.loc "docker logs --tail 50 ecstatic_franklin"

# Restart a container
ssh rio@spark.garage.loc "docker restart ecstatic_franklin"
```

### 7.4 — Proxmox / Nebo Maintenance

```bash
# Update Proxmox packages
ssh root@nebo.garage.loc "apt update && apt upgrade -y"

# VM backup (VM 100 rock)
ssh root@nebo.garage.loc "vzdump 100 --compress zstd --storage datastore1"

# LXC backup (if rag-vm exists)
ssh root@nebo.garage.loc "pct snapshot 200 rag-backup"

# Check Proxmox storage
ssh root@nebo.garage.loc "pvesm status"
ssh root@nebo.garage.loc "df -h"
```

### 7.5 — Common Issues & Resolutions

| Problem | Likely Cause | Resolution |
|---|---|---|
| vLLM OOM at load time | `--gpu-memory-utilization` too high | Lower to 0.35, reduce `--max-model-len` |
| Slow response times | High queue depth, insufficient KV cache | Increase KV cache fraction, add more sequences |
| GPU temp > 80°C | Sustained heavy load | Check airflow, reduce batch size |
| Docker container exits | OOM killer or model error | `docker logs`, check RAM, increase system swap |
| Proxmox API 401 | Auth token expired | Re-authenticate via `chuck.py proxmox` |
| ChromaDB connection refused | rag-vm not running | `ssh root@nebo.garage.loc "pct start 200"` |
| Mac model too large for 16 GB | Swap thrashing | Use smaller quantized model (Q4 ≤ 4B params) |
| Spark unreachable | Network down / node reboot | Check power, `ping spark.garage.loc` |

### 7.6 — Scheduled Monitoring (Optional)

```bash
# Add to crontab on nuzi for regular health checks
# Every 30 minutes:
*/30 * * * * python3 ~/.hermes/scripts/chuck.py check >> ~/AI_Lab_Docs/chuck.log 2>&1
```

### 7.7 — Emergency Procedures

```bash
# === Full Spark Reboot ===
# WARNING: Stops all vLLM inference!
ssh rio@spark.garage.loc "sudo reboot"

# === Kill All GPU Processes ===
ssh rio@spark.garage.loc "pkill -9 -f vllm || true"
ssh rio@spark.garage.loc "nvidia-smi --gpu-reset"  # if GPU is stuck

# === Proxmox VM Rescue ===
ssh root@nebo.garage.loc "qm stop 100"
ssh root@nebo.garage.loc "qm start 100"

# === Network Diagnostics ===
ping spark.garage.loc
ping -c 3 10.20.20.100
traceroute spark.garage.loc
```

---

## Appendix A — Quick Reference Card

```
┌──────────────────────────────────────────────────────────────┐
│  SPARK  (GPU Inference)    ssh rio@spark.garage.loc          │
│  ─────────────────────────────────────────────────────────── │
│  Primary (35B):  http://spark:8000/v1/models                 │
│  Secondary (7B): http://spark:8002/v1/models                 │
│  GPU: nvidia-smi | docker ps | docker stats                  │
├──────────────────────────────────────────────────────────────┤
│  NEBO   (Proxmox + RAG)    ssh root@nebo.garage.loc          │
│  ─────────────────────────────────────────────────────────── │
│  VM 100 rock:  10.20.20.11 | Portainer: :9443                │
│  LXC 200 rag:  ChromaDB host                                 │
│  Proxmox:     https://nebo.garage.loc:8006                   │
├──────────────────────────────────────────────────────────────┤
│  NUZI   (Mac Dev)          ssh rio@nuzi.garage.loc           │
│  ─────────────────────────────────────────────────────────── │
│  llama.cpp (Metal) / MLX / Ollama                            │
│  Chuck agent: python3 ~/.hermes/scripts/chuck.py             │
└──────────────────────────────────────────────────────────────┘
```

## Appendix B — SSH Key & Auth

| Host | User | Auth Method | Notes |
|---|---|---|---|
| spark.garage.loc | `rio` | SSH key (`~/.ssh/id_ed25519`) | Default shell |
| nebo.garage.loc | `root` | SSH key + Proxmox API ticket | API password: deep data 941 |
| nuzi.garage.loc | `rio` | SSH key (`~/.ssh/id_ed25519`) | Default shell |

## Appendix C — Chuck Agent Commands

```
python3 ~/.hermes/scripts/chuck.py check        # Full health check all nodes
python3 ~/.hermes/scripts/chuck.py models       # List vLLM models on spark
python3 ~/.hermes/scripts/chuck.py gpu          # GPU memory on spark
python3 ~/.hermes/scripts/chuck.py proxmox      # Proxmox infrastructure summary
python3 ~/.hermes/scripts/chuck.py vm-list      # List all Proxmox VMs
python3 ~/.hermes/scripts/chuck.py vm-status 100 # Detailed VM status
```

---

*Document generated by Chuck — AI Infrastructure Architect*  
*Last updated: 2026-06-20*