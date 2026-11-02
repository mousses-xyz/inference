# deepseek-v4.1-flash kv cache on omv smb and redqueen k3s placement plan

## tainted-node onboarding gate

redqueen stays outside k3s until the host/runtime, reboot-recovery, and mixed-load acceptance gates in the [AI inference plan](https://github.com/petermousses/kubernetes/blob/d4d5ed1ccc353359f4f8a10b1be8836d724146f2/AI_INFERENCE_PLAN.md) pass. when admitted, it joins as a **k3s agent** for host-managed inference support; do not deploy GPU Operator, claim GPU resources, or schedule inference engines in Pods.

do not join or enable redqueen or other additional nodes during NAS-backed volume migration stages 1–3. finish those stages and obtain the NAS plan's explicit approval before making nodes available. the `litellm` backup/restore row is still pending: complete and verify its backup and isolated restore before moving the gateway. after stage 3, moving each workload still needs its own explicit approval; this plan proposes LiteLLM as a follow-on and does not itself grant that approval. see the [NAS migration plan](https://github.com/petermousses/kubernetes/blob/d4d5ed1ccc353359f4f8a10b1be8836d724146f2/nas-backed-volume-migration-plan.md).

**2026-11-02 waiver of the `litellm` backup precondition.** the owner directed the gateway move while the `litellm` backup/restore row remains pending, inverting the clause above. this is accepted because the gateway holds no state: model configuration is a ConfigMap, and virtual keys, budgets, and spend logs live in PostgreSQL, which does not move. relocating the Pod therefore cannot endanger the database, and rollback is a node-selector revert with no restore step. the `litellm` backup/restore row remains an open NAS-plan obligation and is now decoupled from gateway placement. the node-join gate was waived separately on the same date.

**2026-11-02 preconditions verified before the move.** redqueen is a `Ready` agent carrying `ai.mousses.xyz/accelerator=strix-halo`, `node-role.kubernetes.io/ai=true`, and `ai.mousses.xyz/inference=true:NoSchedule`. cross-node pod routing works in both directions (OMV Pod to redqueen Pod `10.42.1.2:8080`, redqueen Pod to CoreDNS `10.42.0.12:53` on OMV), ClusterIP DNAT is programmed on redqueen (`10.43.0.1:443`), redqueen Pods reach the inference adapters at `10.9.20.242:8081` through the existing firewall allowlist, and the gateway's own service account opens `postgres.litellm.svc.cluster.local:5432` under `allow-database-ingress`. `node-exporter` runs on the node and `/etc/hosts` was restored as a host fix. see [`k3s-agent-onboarding.md`](k3s-agent-onboarding.md).

after those gates and approval, label and taint the joined agent as defined by the AI inference plan:

```sh
kubectl label node <redqueen-node> \
  ai.mousses.xyz/accelerator=strix-halo \
  node-role.kubernetes.io/ai=true
kubectl taint node <redqueen-node> \
  ai.mousses.xyz/inference=true:NoSchedule
```

only explicitly approved monitoring agents and, after its separate move approval, the LiteLLM gateway may tolerate this taint. tolerations permit scheduling but do not select redqueen, so LiteLLM also needs required node affinity or a node selector for the AI label. keep ordinary workloads repelled.

## status and scope

this is a proposal, not a deployment record. it does not authorize joining redqueen, creating an smb share, changing host services, moving the litellm pod, or changing the database.

goals:

1. reuse validated DeepSeek-V4.1-Flash-1017 prompt state from an SMB share hosted by OpenMediaVault (OMV), without making inference depend on the share being available;
2. after the existing migration gates pass, join redqueen as a tainted k3s agent and move only the LiteLLM gateway pod there;
3. keep llama.cpp, llama-swap, model weights, and GPU admission host-managed. keep LiteLLM PostgreSQL on OMV until a separately approved database move.

the test target is the llama.cpp worker `deepseek-v4.1-flash-1017-unsloth`, because `--slot-save-path` and `/slots` are llama-server contracts. the native `deepseek-hip` worker is a separate backend with its own session-buffer implementation; it is out of scope until it exposes an equivalent documented state-file contract.

## existing plan integration

the source plans were checked in the sibling `../kubernetes` checkout at commit [`d4d5ed1`](https://github.com/petermousses/kubernetes/tree/d4d5ed1ccc353359f4f8a10b1be8836d724146f2):

- [`AI_INFERENCE_PLAN.md`](https://github.com/petermousses/kubernetes/blob/d4d5ed1ccc353359f4f8a10b1be8836d724146f2/AI_INFERENCE_PLAN.md) defines redqueen's labels and `ai.mousses.xyz/inference=true:NoSchedule` taint. it keeps inference engines host-managed and describes joining redqueen only after host, mixed-load, and reboot gates. its current v1 architecture pins LiteLLM and PostgreSQL to OMV.
- [`nas-backed-volume-migration-plan.md`](https://github.com/petermousses/kubernetes/blob/d4d5ed1ccc353359f4f8a10b1be8836d724146f2/nas-backed-volume-migration-plan.md) forbids joining or enabling other nodes during backup stages 1–3. after stage 3, node availability and each workload move still require explicit approval. the `litellm` backup/restore row is still pending in that plan.
- the existing [`litellm` Deployment](https://github.com/petermousses/kubernetes/blob/d4d5ed1ccc353359f4f8a10b1be8836d724146f2/apps/litellm/deployment.yaml) is one `Recreate` replica requesting/limited to 1 CPU and 4 GiB, currently pinned to `openmediavault`. the PostgreSQL StatefulSet and migration Job are also pinned there; the database uses a retained local PV. the gateway's cache, migrations, and `/tmp` are disposable `emptyDir`s.

**placement change:** neither existing plan currently specifies moving LiteLLM to redqueen. treat that as a post-v1 follow-on proposal. this plan adds an exact LiteLLM toleration and node placement only after the NAS and Redqueen gates; it does not relax the taint for general workloads. Kubernetes tolerations permit placement but do not select a node, so the gateway also needs required node affinity or a node selector. [Kubernetes taints and tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)

## kv sizing and expected benefit

redqueen's tracked DeepSeek launch is `deepseek-v4.1-flash-1017-unsloth` in [`config.yaml`](model-router/config.yaml): the 7-shard UD-Q2_K_XL GGUF with the BF16 projector, a 220,000-token context, `q8_0` K/V cache types, and the `llama.cpp-d812350-deepseek41-moe-v17` runtime.

DeepSeek-V4.1-Flash is not a plain per-layer transformer cache. the tracked checkpoint's `config.json` and GGUF metadata describe 40 layers, MLA with one KV head and a 512-value latent plus a 64-value rope, `compress_ratios` that select 2:1 and 1:1 compression by layer, and only four **KV source layers** (`[2, 8, 14, 20]`) that own compressed rows. later layers alias their source through the patched `llama_kv_cache_dsv4` reuse callback. index keys come from eight source layers (`[2, 8, 14, 20, 24, 28, 32, 36]`) with 32 heads of 128 values, pooled at 8-token candidate blocks and selected with top-512. a 128-token sliding window covers the raw stream, and the three `nextn`/dspark layers are excluded from the main cache.

the patched runtime sizes the compressed cache for the finest ratio (one row per token), so the token-dependent payload is the four source layers plus the index streams, not all 40 layers. the allocation-free planner the scheduler already uses reports the live context allocation on the pinned runtime:

| context | planner context estimate |
| ---: | ---: |
| 1,000 tokens | 8 MiB |
| 16,000 tokens | 47 MiB |
| 64,000 tokens | 171 MiB |
| 220,000 tokens | 576 MiB |

the check is reproducible on the pinned runtime; the planner's second column is the context allocation:

```sh
/srv/ai/runtimes/llama.cpp-d812350-deepseek41-moe-v17/bin/llama-fit-params --fit-print on \
  --model /srv/ai/models/deepseek-v4.1-flash-1017-unsloth/UD-Q2_K_XL/DeepSeek-V4.1-Flash-1017-UD-Q2_K_XL-00001-of-00007.gguf \
  --ctx-size 220000 --cache-type-k q8_0 --cache-type-v q8_0 --gpu-layers 999 \
  --parallel 1 --batch-size 32 --ubatch-size 1 --split-mode layer \
  --moe-stream --moe-stream-direct --moe-stream-cache 4g --moe-stream-cache-max 120g \
  --moe-stream-io-threads 1 --load-mode none --lazy-mode on-direct --flash-attn auto
```

that is about 2.7 KB per token at long context, or roughly 0.56 GiB per full 220,000-token session. a thousand full sessions would be about 562 GiB (604 GB) before fixed per-session compressor state, sliding-window rows, slot metadata, and filesystem overhead.

these are planner estimates for the live allocation, not measured slot files. measure actual files on the pinned runtime before setting a quota. do not plan for 1,000 full-context sessions without explicit OMV capacity and retention approval.

cache reuse only avoids recomputing a matching prefix. it does not speed the first cold prefill, new suffix tokens, model loading, or token generation. restoring a state can resume directly at the next token only when the next request adds no prompt tokens; a normal chat turn must prefill its newly appended user message and chat-template suffix.

## phase 1 — prove the pinned runtime can restore DeepSeek state

do this on redqueen with the exact deployed DeepSeek GGUF, tokenizer/chat template, and `llama.cpp-d812350-deepseek41-moe-v17` runtime before provisioning the SMB cache. the general `llama-server` slot API supports `--slot-save-path` in documented versions, but the live redqueen binary and its custom patches must be checked directly. [llama-server slot API](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md?plain=1)

the pinned runtime already exposes `--slot-save-path`, `--cache-ram` (8 GiB default), `--cache-idle-slots` (enabled by default), and `--ctx-checkpoints` (32 default). its custom `llama_kv_cache_dsv4` implements `state_write`/`state_read`, but that is source-level support, not proof that a saved slot reuses tokens.

1. check the pinned server's `--help` and local `/slots` API for save/restore support; record the exact runtime revision and patch-manifest hash.
2. with a disposable local directory, run a long DeepSeek prompt, save its slot, erase/unload it, restore it, then resend the exact token prefix plus a short suffix.
3. verify real prefix reuse from processed/cached token counts and TTFT. `n_restored` or HTTP 200 alone is insufficient.
4. repeat after a server restart and with the longest expected prompt. exercise failed, truncated, incompatible, and missing state files.
5. compare `--slot-save-path` against the in-memory prompt cache (`--cache-ram`/`--cache-idle-slots`). the in-memory checkpoints are not durable and must not be counted as SMB restores.

**hard gate:** upstream llama.cpp has reported that slot save/restore on hybrid/recurrent models can return success but reprocess the full prompt because server prompt checkpoints are not persisted. DeepSeek-V4.1's cross-layer KV aliasing and compress ratios make this directly relevant. the redqueen source currently has no configured `--slot-save-path`; do not enable disk persistence until the exact pinned build proves a cache hit. if it fails, either track and test a compatible checkpoint-persistence fix in the pinned runtime or stop this project at the in-memory cache. do not solve this by silently upgrading the production runtime. [upstream hybrid restore issue](https://github.com/ggml-org/llama.cpp/issues/25913)

## phase 2 — provision an isolated OMV SMB cache share

after phase 1 passes **and** NAS backup stages 1–3 and their approval gates are complete:

1. create a dedicated OMV directory and SMB share for DeepSeek cache files, separate from PostgreSQL, user files, model weights, and Borg/Restic backup repositories. proposed filesystem path: `/filesystem/k3s/data/redqueen-kv-cache/deepseek-v4.1-flash-1017/`; verify the actual OMV filesystem, free space, quota support, and snapshot/backup policy first.
2. create one non-admin SMB identity for redqueen cache access. grant it read/write only on this share; never commit its credential or put it in a command line, environment dump, or model-router config.
3. restrict TCP 445 to redqueen's reserved host address (`10.9.20.242`) on OMV and redqueen firewalls. require SMB 3.1.1 and SMB-layer encryption if the OMV server supports it. store a root-owned, mode-0600 credentials file outside the checkout. [mount.cifs options](https://www.man7.org/linux/man-pages/man8/mount.cifs.8.html)
4. mount the share on redqueen at `/srv/ai/model-router/cache/kv/deepseek-v4.1-flash-1017`, owned by the `ai` service identity with restrictive modes. use a root-managed systemd mount/automount with network-mount dependencies; verify mount identity before every cache operation so an absent share can never write into the local mountpoint. [systemd mount units](https://freedesktop.org/software/systemd/man/latest/systemd.mount.html)
5. test create/read/rename/fsync, a multi-gigabyte sequential transfer, reconnect after OMV restart, and read-only/full-share failure. gate every save/restore call on a verified CIFS mount; when SMB is down, skip cache I/O and run the ordinary inference path without persistent caching. never let a cache operation fall through to the local mountpoint directory.
6. exclude this disposable cache share from routine backups and snapshots unless the owner explicitly chooses otherwise. enforce a byte budget, maximum sessions, and age/LRU eviction before production use. record that eviction removes performance state, not recoverable model or user data.

## phase 3 — add safe session-aware cache orchestration

`--slot-save-path` exposes state save/restore operations; it does not by itself provide a durable conversation-to-slot policy. LiteLLM must not be assumed to provide the required stable session identity or lifecycle callbacks without a prototype.

1. prototype a host-local cache broker adjacent to the existing scheduler/llama-swap path. it must acquire the existing DeepSeek worker lease, serialize access to the single `--parallel 1` slot, restore before dispatch, and save only after successful generation completion and backend drain. the worker's `concurrencyLimit: 2` does not create a second KV slot. a client disconnect must not release the lease before the backend stops using the slot.
2. accept a stable conversation/session identifier from the trusted caller when available. otherwise key entries from the canonical token prefix produced by the runtime's DeepSeek-V4.1 prompt encoding and the GGUF chat template. always verify the saved token hash and model identity before reuse; never use a caller-controlled filename.
3. namespace entries by model-file manifest/checksum, tokenizer and chat-template revision, llama.cpp revision/patches, cache quantization, and cache-format version. invalidate incompatible entries and fall back to full prefill.
4. keep slot save/restore endpoints private to the host broker. do not expose them through LiteLLM, the public API, or a cluster Service. avoid logging prompts, tokens, cache contents, or raw session IDs.
5. use one writer per session; write a temporary file on the same SMB share, verify it, then atomically rename it. incomplete or corrupt files are misses. a save failure must not turn a completed inference into a failed response.
6. ensure a restored response is bound to the same authenticated LiteLLM virtual-key/session scope. do not share KV state across users merely because their system prompts match.

benchmark cold prefill, warm in-memory prefix reuse, and SMB restore separately at representative 16k, 64k, and 220k token contexts. record restore/save duration, bytes transferred, prefix tokens reused, suffix tokens processed, TTFT, and decode rate. require SMB restore to beat cold prefill by a useful margin on the actual network. test interruption during save, missing share, out-of-space, client cancellation, concurrent sessions, and server restart. a restored slot must never produce a different continuation from a cold run on the same exact prompt and decoding settings.

## phase 4 — satisfy cluster and migration gates

the onboarding and NAS sequencing gates at the beginning of this plan are mandatory. before joining, validate the K3s agent version, Redqueen node IP/hostname, cluster join path, Pod-CIDR routing, firewall rules, node allocatable resources, and monitoring agents. transfer the agent join token through a private owner-managed path; never put it in Git. reserve the LiteLLM Pod's actual CPU and memory use in Redqueen's host budget: its current Kubernetes request and limit are 1 CPU and 4 GiB, in addition to the K3s agent and host scheduler. after approval, join redqueen as an agent and apply the labels and taint defined in the opening section. keep raw host API ports private to the cluster/host firewall allowlist. [`k3s-agent-onboarding.md`](k3s-agent-onboarding.md) records the observed host state and the exact owner-run commands.

## phase 5 — move the LiteLLM gateway, not its database

make the placement change in the sibling Kubernetes repository, not in this inference repository. keep the current one-replica `Recreate` strategy and the existing service, public IngressRoute, virtual keys, and model configuration.

1. change only the LiteLLM gateway Deployment from the `openmediavault` node selector to required redqueen node placement, and add the exact inference-taint toleration. keep UID/GID, read-only root filesystem, resource limits, probes, service account policy, Secrets, and security controls unchanged.
2. keep LiteLLM PostgreSQL and its retained local PV on OMV. keep the short-lived LiteLLM migration Job on OMV unless a separately reviewed change requires otherwise. do not copy or relocate PostgreSQL data as part of this gateway move.
3. preserve the LiteLLM ClusterIP and Traefik routes. prove from a Pod on redqueen that PostgreSQL, Authentik/OIDC, Redqueen inference adapters, image endpoints, and required DNS/policy paths work. verify actual source addresses at redqueen's firewall before modifying any allowlist.
4. use `./apps/litellm/deploy.sh` for any LiteLLM release/schema change; never deploy this application with plain `kubectl apply -k apps/litellm`. a placement-only cutover must not run an unreviewed schema migration.
5. accept the planned `Recreate` downtime explicitly. validate authenticated models, streaming, long DeepSeek first-token behavior, OIDC/Admin UI, metrics scraping, database connectivity, network policies, and the existing 30-minute mixed workload. check Redqueen memory pressure, scheduler admission, GPU resets, OOM counters, and service restarts.
6. confirm only the LiteLLM gateway Pod is on redqueen; PostgreSQL and the migration Job remain on OMV. confirm no ordinary cluster workload can schedule on the tainted node.

this placement is not HA: OMV remains the k3s control plane, public ingress host, and LiteLLM database host. a later database/cluster-HA project must follow the separate approval and restore-based migration sequence in `nas-backed-volume-migration-plan.md`.

## rollback and acceptance

**kv rollback:** disable broker restore/save and the disk-cache flag; inference falls back to normal prefill. keep cache files until the owner-approved retention window expires. do not restore files across changed weights, tokenizer/template, runtime, or cache format.

**LiteLLM rollback:** restore the gateway's OMV node selector and remove its redqueen toleration, then roll out the existing Deployment and verify the same service/API path. the database never moved, so no database restore is part of this rollback. retain the Redqueen node taint and approved monitoring tolerations.

accept only when the pinned DeepSeek runtime demonstrates real prefix-token reuse after SMB restore, cache failures degrade to misses without interrupting inference, privacy/capacity limits are enforced, and the LiteLLM relocation passes every phase-5 check. until then, disk KV caching remains disabled and LiteLLM remains on OMV.
