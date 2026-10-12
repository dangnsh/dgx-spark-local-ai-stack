# Reddit posts (link back to the full writeup)

Full writeup for all three: https://github.com/dangnsh/dgx-spark-local-ai-stack

---

## 1) r/LocalLLaMA — title:

Two DGX Sparks running one 180B MoE as an always-on local assistant (vLLM TP2, memory, self-heal, nightly "dream")

## Body:

I run a Hermes agent as a personal assistant on two DGX Sparks at home. Not a
demo that runs when someone's watching: it handles email triage, family
calendars, reminders, and image generation, and it has to survive its own bugs
with nobody at the keyboard. Full writeup on GitHub (link at the bottom); the
short version:

- One 180B-A6B MoE brain (Qwen3.8-Flash-Next, NVFP4) served by vLLM
  tensor-parallel across both nodes over the ConnectX-7/RoCE link. It only fits
  at TP2, which is why the cluster exists. Speculative decoding (MTP) on, ~11
  min cold boot.
- GB10 unified memory means CPU and GPU share one 128GB pool, so
  `--gpu-memory-utilization` really budgets host RAM. Almost every nasty bug
  traced back to that: a heavy engine starting while memory is thin holds the
  nvidia driver's global lock and freezes the entire SoC. The anti-freeze rules
  and the worker-first boot order (plus the page-cache ritual that took me a
  week to learn) are written up properly in the doc.
- Long-term memory via Hindsight: daemon + Postgres on node 1, two small
  llama.cpp sidecars on node 2 so they never fight the brain for RAM. Fun
  fact: the 4B extractor runs faster CPU-only (20 tok/s) than partially
  offloaded (16.6) on this hardware.
- Self-heal is thirteen crontab lines and a pile of shell scripts. The lesson
  that cost the most: /health returning 200 does NOT mean the brain is alive;
  a dead rank lets the head answer health checks for hours. Every watchdog now
  sends a real completion request.
- A nightly 03:30 "dream" job: the agent reads its own transcripts from the day
  and writes a review, auto-applying confident fixes and queueing the rest as
  proposals for me. Output is a file + a message, never injected back into the
  prompt.
- Film mode: the model's n-gram table gets mmap'd from NVMe so the brain fits
  on ONE Spark (~76GB resident), and node 2 becomes a dedicated ComfyUI render
  farm. Same endpoint name, so the agent's config doesn't change when I flip
  modes. If I only had one Spark, this is what I'd run.

Ask me anything: vLLM flags on GB10, RoCE setup, watchdog design, the dream
job's rules. I'd rather answer ten questions than watch someone else spend a
weekend on the page-cache thing.

Full writeup: https://github.com/dangnsh/dgx-spark-local-ai-stack

---

## 2) r/hermesagent — title:

What running Hermes Agent against a 2-node local cluster actually looks like (memory, self-heal, nightly dream)

## Body:

Most posts here are about models or skills, so I wanted to write up the other
half: what the agent sits ON. I run Hermes against two DGX Sparks (GB10) as
our family assistant: Telegram for me, WhatsApp for my wife, email triage,
calendars, reminders, the works. The full writeup is on GitHub (link below);
the Hermes-specific parts:

- Model endpoints are just provider entries in config, re-read every turn, with
  an ordered fallback chain. TP2 180B engine -> single-node engine -> 27B on
  one box -> cloud. The infra can die in almost any configuration and the agent
  degrades instead of going dark. I cut over engines without touching Hermes
  config at all.
- The gateway multiplexes five agent profiles over one process (family/infra,
  code, media, church), each with its own skills and memory, routed by platform.
- Every scheduled job is a full agent turn with tool access. The weekly
  self-update job is an agent that updates the system and then re-verifies its
  own work like an on-call engineer.
- Nightly "dream" at 03:30: the agent reads its own transcripts from the day,
  patches its own skills when a step demonstrably failed, and DMs me a numbered
  list of everything it's not confident enough to touch. Deliberately never
  injected back into the prompt: the dream writes files, the files shape future
  turns.
- Long-term memory via Hindsight (facts, entity resolution, budgeted recall),
  with the small extraction/embedding models on the second node so they never
  compete with the brain for RAM.

The writeup also covers the infra side (vLLM TP2 over RoCE, the GB10 unified
memory traps, thirteen watchdog crons, single-node fallback mode). Would be
good to compare dream-job designs with what others here are doing.

Full writeup: https://github.com/dangnsh/dgx-spark-local-ai-stack

---

## 3) r/nvidia — title:

Squeezing a 180B MoE across 2 DGX Sparks over RoCE, and giving node 2 its job back

## Body:

A hardware-focused writeup on the two Sparks, from six months of running them
as an always-on stack (the client is a Hermes agent serving my family; the doc
covers that too, link below). The GB10-specific findings:

- No separate VRAM is not a footnote. `nvidia-smi --query-gpu=memory.*` returns
  "Not Supported", vLLM's GMU budgets host RAM, and a heavy engine starting
  under memory pressure holds the nvidia driver's global lock while the whole
  SoC freezes: desktop, nvidia-smi, everything. Hard reboot. The rules that
  stopped that from happening: one heavy engine per node, ~40GB headroom gate
  before any load, and no `--enable-torch-compile` (the 9-minute CUDA graph
  capture is the lock-holder).
- TP2 over the ConnectX-7 pair works, but boot choreography matters: drop page
  cache on both nodes, start the worker rank first, head ~25 s later. Reverse
  it and you get a rank desync that only a full stop/start fixes. Mixed kernel
  versions across the two boxes can also silently break NCCL after one node
  updates; boot the old kernel before touching the distributed setup.
- A dead worker rank lets the head answer /health for hours. The NCCL failure
  only surfaces at the next collective. Any real TP health check must read the
  worker's container state too.
- CPU-only llama.cpp sidecars beat partial offload on this box (measured 20 vs
  16.6 tok/s on the same 4B). Weights live in page cache, which is the cheapest
  memory on a unified design.
- Node 2 duty shift: an NVFP4 180B with its ~48GB n-gram table mmap'd from
  NVMe (~76GB resident) fits on ONE Spark, so a toggle moves the brain to
  single-node and node 2 becomes a dedicated ComfyUI render farm. Same endpoint
  name; the client agent never notices.

Full writeup with all the details and the watchdog list:
https://github.com/dangnsh/dgx-spark-local-ai-stack
