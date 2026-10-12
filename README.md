# Two DGX Sparks, one always-on local AI stack

I run a personal AI assistant on two NVIDIA DGX Sparks at home. Not a demo that
runs a chatbot when someone's watching: it handles email triage, family
calendars, reminders, church bulletins, image generation, and it's supposed to
survive its own bugs without me. This post is how the whole thing is wired:
tensor parallel across the two nodes, long-term memory, self-healing watchdogs,
a nightly "dream" job, and the single-node mode I use when I want the second
box back for something else.

## The hardware

Two DGX Sparks. Each one is a GB10 Grace Blackwell: 20-core Arm CPU, 128GB of
unified memory that the CPU and GPU share, so `free` and the GPU are looking at
the same pool. The two boxes are tied together with their ConnectX-7 ports
over fiber, running RoCE, which is what makes tensor parallel across chassis
actually usable. DGX OS 7.2.3 on both, ARM64 userland.

The single most important thing to internalize about this hardware: there is no
separate VRAM. `nvidia-smi --query-gpu=memory.*` literally returns "Not
Supported". If a model and a browser and a background job all want RAM, they
fight over one number. Almost every nasty bug I've had traces back to that.

## What runs where

Right now, in "brain mode":

- **spark1 (the head).** The agent gateway, the memory daemon and its Postgres,
  and rank 0 of the vLLM engine. This is also the box the agent itself lives
  on, which turns out to have interesting consequences (see the lessons).
- **spark2.** Rank 1 of the same vLLM engine, plus two small sidecar models for
  the memory system, plus a Postgres container for a dashboard app.

One model serves everything: a Qwen3.8-Flash-Next build, NVFP4 quantized,
served by vLLM with tensor parallelism across both nodes, listening on one
endpoint on spark1. Speculative decoding (MTP) is on. Boot to healthy is about
11 minutes, most of it weight loading.

Why tensor parallel instead of running two independent models? Because the
model I actually want as my assistant doesn't fit in 128GB after quantization,
and 242GB combined does. The RoCE link hides the split well enough for chat
workloads; a 200-token smoke test comes back in seconds.

The choreography that boot requires is worth writing down because it bit me:
drop the page cache on *both* nodes first (`sync; echo 3 >
/proc/sys/vm/drop_caches`), start the *worker* rank, wait ~25 seconds, start
the head. Reverse the order and you get a rank desync that no amount of
restarting the head will fix; you have to stop both and start over. The mmap'd
checkpoint shards pin whatever page cache the download flooded in, RAM
collapses mid-load, and the worker dies with no traceback. vLLM just says
"WorkerProc initialization failed" and you waste an afternoon blaming software.

## Memory management, the model kind and the actual-RAM kind

vLLM's `--gpu-memory-utilization` on unified memory is a lie of a flag name: it
budgets *host RAM*. Set it too high and the colocated services on the same box
start dying with generic connection errors while the model server looks
perfectly healthy. It took me embarrassingly long to connect saturated swap in
`free -h` with subagent failures, because the obvious suspect (the model
endpoint) always returned 200.

So the rules now, in order of how much pain they cost me:

1. Never two heavy LLM engines on one node. At all. Ever.
2. Never start an engine when available RAM is under ~40GB; skip the cycle and
   retry later instead of forcing the load.
3. No auto-start of the serving engine at boot; a boot script does it, but
   gated on headroom.
4. Drop caches before loading any big model.
5. `--enable-torch-compile` stays off. Startup becomes a 9-minute CUDA graph
   capture, and on GB10 a long startup under memory pressure means the driver's
   global lock is held for minutes while everything else blocks. The whole SoC
   freezes: desktop, nvidia-smi, the gateway, all of it. Hard reboot. That one
   failure mode is why the rules above are religion rather than suggestions.

## Long-term memory (Hindsight)

The agent talks to a memory daemon on spark1 (HTTP API + embedded Postgres).
Every conversation turn is retained asynchronously, extracted into structured
facts, entities get resolved, and relevant memories are injected into future
turns with a budget. Two small models do the work and they live on spark2 so
they never compete with the brain for the same gigabytes:

- an embedding server (Qwen3-Embedding-0.6B, 1024-dim, CPU-only)
- an extraction LLM (a 4B model) that turns raw turns into memory units

CPU-only here is not a downgrade story. I measured the same 4B model at 20
tok/s with zero GPU offload versus 16.6 tok/s partially offloaded on GB10; the
offload pays more in transfer overhead than it earns. The weights live in page
cache, which on this box is about as close to free as memory gets.

The trap with llama.cpp sidecars: they allocate to the high-water mark of the
biggest batch they've ever served and never give it back. A 0.6B model was
holding 3.7GB after a week. A cron job on the head checks every 7 minutes and
restarts them (3s and 7s to reload, data stays in Postgres) when their combined
anon RSS crosses 4GB *and* the endpoints are idle *and* the node has headroom.
Three conditions because a restart during a load spike once dipped node RAM
enough for the memory guard to kill the brain's worker rank.

## Self-heal

Everything that's supposed to recover without me is a cron line and a shell
script. No Kubernetes, no fancy orchestration. Thirteen crontab entries, a few
of which overlap on purpose:

| Job | Every | Does |
|---|---|---|
| memguard | 1 min | kills the serving container when RAM is genuinely critical |
| brain-watchdog | 2 min | real chat request against the endpoint; relaunches if dead |
| n2-recover | 3 min | restarts the worker rank if it vanished |
| hindsight-watchdog | 5 min | restarts the memory daemon + DB if the port is dark |
| hindsight-restart | hourly-ish | condition-based sidecar RAM reclaim (above) |
| gpu_thermal | 1 min | logs clocks/temps, throttling-visible |
| selfheal-head | @reboot | brings the whole stack back in order after power loss |

The details that matter more than the list:

**A 200 on /health is not a brain.** When the worker rank dies, the head keeps
serving health checks for hours. It only finds out at the next collective op,
NCCL throws a retry-exceeded, and the engine self-shutdowns politely with exit
code 0. Every probe in the house now sends a small real completion request, not
just a port check.

**Watchdogs are a double-edged sword and need a kill switch.** Every
auto-starter honors a flag file (`film-mode.on`, `film.pref`), because
otherwise three different watchdogs will happily resurrect the brain onto the
render node mid-render and freeze it. Related: never "restore" a crontab with
`crontab -l | grep ... | crontab -`. Piping into `crontab -` *replaces the
whole crontab*. I lost every guard for 26 minutes to a clever one-liner.
Back up, edit a temp copy, install the file, count the lines afterward.

**The upgrade that kills the brain is the one that looks safe.** An apt
auto-update classified docker + nvidia-container-toolkit as "no reboot
required". Upgrading docker restarts dockerd, which stops every container,
including the head rank. The worker notices 30 minutes later. And the cron job
running the upgrade was itself a call through the dying brain, so the failure
arrived in chat as "model: Connection error". Those packages are now
maintenance-window-only, hardcoded into the update job's deny list.

**earlyoom** runs on both nodes with the crown-jewel processes excluded
(sshd, dockerd, postgres, the llama sidecars). memguard is the surgical knife,
earlyoom is the blunt one that only needs to fire once a year.

There's also an external Raspberry Pi in the plans (half-built) whose only job
is to poke these endpoints from a different power socket and shout on Telegram
when they don't answer. A cluster can't be trusted to report its own death.

## The nightly dream

At 03:30 the agent reads its own transcripts from the day (raw conversation
documents, not derived memories) and writes a review: procedures it got wrong,
preferences the humans repeated, facts in its brain that are now stale. Two
modes, currently on `auto`: confident fixes (a skill step that demonstrably
failed, a superseded memory unit) get applied by itself with a revert note;
anything touching infra, rules, or judgment calls goes into a morning message
as a numbered proposal. The output is a file plus a DM. It is deliberately
never injected back into the prompt; the nightly reflection informs the files,
the files inform the next turn. A quiet day with "nothing found" is a valid
result, and the job is instructed so, because an LLM asked to review will
always invent findings to look busy. There's a monthly grooming pass too,
which dedupes and soft-invalidates old memory units.

Alongside the dream, cron carries the actual agent workload: morning briefings,
email triage, a pile of small one-shot reminders, a weekly auto-update that
knows to re-verify everything after itself. The scheduler re-reads its job
file every tick, so schedule edits land live.

## The other mode: one node for the brain, one node for everything else

Tensor parallel works right up until you want the second box to itself. Video
and image work (ComfyUI, LoRA training) is single-GPU by nature and can't be
split across chassis, so the only real split is job-level: which node does
what.

So there's a film mode. The trick that makes the brain fit on one Spark: the model's big
n-gram table (~48GB of it) gets memory-mapped from NVMe instead of held
resident (`VLLM_PLE_MMAP=1`, plus a splitting op in a piecewise CUDA graph).
Resident weights drop to ~76GB at 0.80 GMU, fits, and spark2 boots the brain
worker out entirely and becomes a pure render farm with ComfyUI on it. Same
port, same served model name, so the agent's config doesn't change at all when
I flip `film-toggle.sh film` and back. Decode is slower (NVMe reads instead of
a second GPU), which is the honest price of freeing the node.

So: if you have two Sparks and the model only fits at TP2, you're not locked
into that. An mmap'd table, one container variant, and a handful of flag-gated
watchdogs later, node 2 can spend its life generating images instead of
holding KV cache. If I only had one Spark, this single-node variant with the
same mmap trick is what I'd run as the daily brain, and it's good enough that
I keep it as the fallback when the TP2 pair is down anyway.

## Things I'd tell myself six months ago

- The freeze that hard-reboots your box is almost always memory, held behind a
  driver lock, caused by starting the new engine before the old one died.
- Correlate timestamps before blaming: a failed job is usually the victim of
  a silent rank loss, not the cause of it.
- `grep` your own pkill patterns into oblivion: `pkill -f foo` matches the
  shell that runs it. Bracket one letter (`[f]oo`) or kill by pidfile.
- A watchdog whose script has a syntax error looks installed and is dead.
  `bash -n` your guards like it's CI.
- Mixed kernel versions across the two nodes can work fine right up until they
  don't; when NCCL breaks after one node updates, boot the old kernel before
  you touch the distributed setup.
- Measure before moving services. Twice I proposed relocating a sidecar "to
  free RAM" and a real per-process census showed both proposals would have
  made things worse.

## Numbers, ballpark

- 2x GB10, 242GB usable unified memory, ConnectX-7 RoCE between nodes
- Brain: 180B-A6B MoE, NVFP4, vLLM TP2, ~11 min cold boot, healthy within
  seconds after that
- Restarting a sidecar model: 3-7 seconds. Restarting my patience: longer.
- Self-heal coverage: the stack has survived kernel mishits, a dockerd
  upgrade, several rank deaths and at least one full-node freeze, all while
  nobody was at a keyboard.

Ask me anything; happy to go deeper on the vLLM flags, the RoCE setup, or the
dream job's prompt design in the comments.
