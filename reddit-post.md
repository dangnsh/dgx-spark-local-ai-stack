# Reddit post body (short version, links to the full GitHub writeup)

Title: Two DGX Sparks, one always-on local AI stack (memory, self-heal, nightly "dream", and how I free up node 2)

Body:

I run a personal AI assistant on two DGX Sparks at home. Not a demo that runs
when someone's watching: it handles email triage, family calendars, reminders,
and image generation, and it has to survive its own bugs without me at the
keyboard.

The full writeup is on GitHub (link below). The short version:

- One 180B-MoE brain (Qwen3.8-Flash-Next, NVFP4) served by vLLM tensor-parallel
  across both nodes over the ConnectX-7/RoCE link. It only fits at TP2, which
  is why the cluster exists.
- GB10 unified memory means CPU and GPU share one 128GB pool. Almost every
  nasty bug traced back to that: a heavy engine starting while memory is thin
  holds the nvidia driver's global lock and freezes the entire SoC. The
  anti-freeze rules, the page-cache ritual, and the worker-first boot order are
  written up properly in the doc.
- Long-term memory via Hindsight (daemon + Postgres on node 1, two small
  llama.cpp sidecars on node 2 so they never fight the brain for RAM). Sidecars
  swell over time (llama.cpp allocates to its high-water mark and never gives
  back), so there's a condition-based reclaim cron for them.
- Self-heal is thirteen crontab lines and a pile of shell scripts: a memory
  guard, brain/worker/memory watchdogs that send a real completion request
  instead of trusting /health (a dead rank lets the head answer health checks
  for hours), thermal logging, and a @reboot script that rebuilds the stack
  after power loss. No k8s. The doc covers the failure modes that shaped each
  guard, including the apt upgrade that restarts dockerd and kills the brain.
- A nightly 03:30 "dream" job: the agent reads its own transcripts from the
  day and writes a review, auto-applying confident fixes and DM'ing the rest
  as numbered proposals for me. Output is a file + a message, never injected
  back into the prompt.
- Film mode: the n-gram table gets mmap'd from NVMe so the brain fits on ONE
  Spark (resident weights ~76GB), and node 2 becomes a dedicated ComfyUI render
  farm. Same endpoint name, so nothing upstream changes when I flip modes.

Ask me anything in the comments: vLLM flags on GB10, the RoCE setup, watchdog
design, the dream job's prompt rules. I'd rather answer ten questions than
watch someone else spend a weekend on the page-cache thing.

Full writeup: <GitHub link here>
