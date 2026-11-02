# redqueen k3s agent onboarding

## status

owner requested onboarding on 2026-11-02. this file records the observed host state and the exact owner-run commands; it is not a record of a completed join.

**gate:** [`nas-backed-volume-migration-plan.md`](https://github.com/petermousses/kubernetes/blob/d4d5ed1ccc353359f4f8a10b1be8836d724146f2/nas-backed-volume-migration-plan.md) and [`kv-cache-smb-and-k3s-plan.md`](kv-cache-smb-and-k3s-plan.md) forbid joining or enabling additional nodes during NAS-backed volume migration stages 1–3. stage 1 is complete. stage 2 still has `librechat`, `litellm`, and `jellyfin` open, and stage 3 has not started. proceeding now **waives that sequencing gate**; record the waiver in the NAS plan before or with the join, or stop here.

**privilege:** `ai` has no passwordless sudo on redqueen, and `peter` is not reachable with the current workstation keys. the install and uninstall commands therefore need a root shell on redqueen. the OMV commands also need the OMV Kubernetes admin account and are listed separately.

do not deploy the GPU Operator, claim GPU resources, or move any inference engine into a Pod. the only planned tenant is the LiteLLM gateway, after its own approval.

## observed preconditions — 2026-11-02

read-only checks over `ssh -i ~/.ssh/redqueen ai@redqueen.mousses.xyz`:

- hostname `amd-halo`; `redqueen.mousses.xyz` resolves to `10.9.20.242`, carried on `enp191s0` (`10.9.20.242/24`);
- vendor OS `rex` 1, kernel `6.18.44+rex+5-amd64`, no swap, cgroup v2 with `cpu`, `memory`, and `pids` controllers;
- 125 GiB RAM, about 115 GiB available before inference; 290 GiB free on `/`;
- `https://10.9.20.14:6443` is reachable from the node and returns `401 Unauthorized` without credentials;
- `/etc/resolv.conf` is the systemd-resolved stub (`127.0.0.53`) with search `mousses.xyz`; k3s normally detects the stub and points Pods at the real upstream file. verify Pod DNS after the join;
- **`/etc/hosts` is absent.** this is invisible to normal host operation but fatal to every Pod sandbox. see [section 5](#5-host-defect-found-during-verification--etchosts-is-missing) and fix it before installing the agent;
- the inference listeners, the `inet redqueen_inference` nftables table, the `ai` user services, and the scheduler are unchanged by this runbook. the nftables table uses `policy accept`, so it does not block k3s; its inference allowlist still covers only `10.9.20.14`, `10.9.20.242`, and `10.42.0.0/16`.

## 1. OMV — capture the join inputs

run as the OMV Kubernetes admin. keep the token out of shell history, chat, and Git.

```sh
k3s --version
kubectl get nodes -o wide
kubectl get node openmediavault -o jsonpath='{.metadata.labels}{"\n"}{.spec.taints}{"\n"}'
sudo cat /var/lib/rancher/k3s/server/node-token
```

record the exact server version (for example `v1.33.5+k3s1`). the agent must match it. do not continue until the version is known.

check the OMV firewall only if its input policy is not `accept`:

```sh
sudo nft list ruleset | grep -E '6443|8472|10250' || true
```

the join needs TCP `6443` from `10.9.20.242` to `10.9.20.14`, UDP `8472` both ways for flannel VXLAN, and TCP `10250` from `10.9.20.14` to `10.9.20.242`. add only the missing rules.

## 2. redqueen — stage the token

in a normal `ai` shell, paste the token at the hidden prompt:

```sh
read -rsp 'node token: ' K3S_TOKEN; echo
printf '%s' "$K3S_TOKEN" | sudo tee /root/k3s-agent-token >/dev/null
sudo chmod 0600 /root/k3s-agent-token
unset K3S_TOKEN
sudo test -s /root/k3s-agent-token && echo token-staged
```

## 3. redqueen — install the agent

replace `<server-version>` with the exact value from step 1.

```sh
curl -sfL https://get.k3s.io | sudo env \
  INSTALL_K3S_VERSION='<server-version>' \
  K3S_URL='https://10.9.20.14:6443' \
  K3S_TOKEN_FILE='/root/k3s-agent-token' \
  sh -s - agent \
    --node-name redqueen \
    --node-ip 10.9.20.242 \
    --flannel-iface enp191s0 \
    --node-label ai.mousses.xyz/accelerator=strix-halo \
    --node-taint ai.mousses.xyz/inference=true:NoSchedule \
    --kubelet-arg=system-reserved=cpu=1,memory=2Gi \
    --kubelet-arg=kube-reserved=cpu=500m,memory=1Gi
```

**do not add `--node-label node-role.kubernetes.io/ai=true` here.** kubelet refuses any `kubernetes.io`/`k8s.io`-namespace label passed through `--node-labels` unless it is one of its hard-coded role names, and it exits immediately with:

```
Error: failed to validate kubelet flags: unknown 'kubernetes.io' or 'k8s.io' labels
specified with --node-labels: [node-role.kubernetes.io/ai]
```

that is fatal to the agent even though the token, CA certificate, and API server discovery already succeeded. apply the role label from OMV with `kubectl` instead (step 4); the restriction applies only to the kubelet flag, not to the API. `ai.mousses.xyz/*` labels and taints are unaffected.

the taint is applied at registration so no ordinary workload can land before it exists. the reservations reduce allocatable rather than pinning a cgroup; no memory eviction threshold is set, because host-managed inference legitimately consumes most of the node's RAM and kubelet must not evict LiteLLM for it.

confirm the service and ports. if an earlier run left the unit crash-looping, just re-run the block above: the installer rewrites `/etc/systemd/system/k3s-agent.service` and its `.env`, resets the unit, and restarts it. no manual unit editing is needed.

```sh
sudo systemctl status k3s-agent --no-pager
sudo journalctl -u k3s-agent -n 80 --no-pager
ip -br addr show cni0 flannel.1
sudo ss -lntup | grep -E ':(10250|6443)\b'
```

## 4. OMV — verify the node and the taint

reapply the labels and taint idempotently, then inspect. this also covers the case where a k3s build ignored the registration flags.

```sh
kubectl label node redqueen ai.mousses.xyz/accelerator=strix-halo --overwrite
kubectl label node redqueen node-role.kubernetes.io/ai=true --overwrite
kubectl taint node redqueen ai.mousses.xyz/inference=true:NoSchedule --overwrite

kubectl get nodes -o wide
kubectl get node redqueen -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}{"\n"}'
kubectl get node redqueen -o jsonpath='{.metadata.labels}{"\n"}{.spec.taints}{"\n"}'
kubectl get pods -A -o wide --field-selector spec.nodeName=redqueen
```

expected: `Ready=True`, `INTERNAL-IP` `10.9.20.242`, the two AI labels, the `NoSchedule` taint, and no *ordinary* Pods. a `flannel` or CNI warning is not enough by itself; confirm the node condition and a Pod network path before calling the join complete.

the pod list will not be empty. cluster-wide monitoring DaemonSets (for example `kube-prometheus-stack-prometheus-node-exporter`) tolerate every taint with `operator: Exists`, and the AI inference plan intends monitoring agents to run here. what must not appear is a normal Deployment, StatefulSet, or Job. any monitoring Pod that appears must reach `Running`; a DaemonSet stuck in `ContainerCreating` on this node is a real defect, not an accepted side effect.

optional Pod DNS check (operator-run diagnostic, not a workload):

```sh
kubectl run redqueen-dns-check --rm -it --restart=Never \
  --image=busybox:1.36 \
  --overrides='{"spec":{"nodeSelector":{"ai.mousses.xyz/accelerator":"strix-halo"},"tolerations":[{"key":"ai.mousses.xyz/inference","operator":"Equal","value":"true","effect":"NoSchedule"}]}}' \
  -- nslookup kubernetes.default
```

delete it if it does not self-remove:

```sh
kubectl delete pod redqueen-dns-check --ignore-not-found
```

then confirm no ordinary workload followed it onto the node:

```sh
kubectl get pods -A -o wide --field-selector spec.nodeName=redqueen
```

## 5. host defect found during verification — `/etc/hosts` is missing

**this blocks every Pod on the node and must be fixed before any workload placement.**

observed 2026-11-02: `/etc/hosts` does not exist on redqueen. `statx /etc/hosts` fails with `ENOENT`; it is not a dangling symlink. the host appears healthy anyway because `/etc/nsswitch.conf` lists `files myhostname ... resolve ...`, so `getent hosts localhost` is answered by `myhostname` rather than by `files`.

containerd's CRI builds each sandbox `/etc/hosts` from the host file, so sandbox creation fails for every Pod regardless of image, toleration, or network mode:

```
failed to generate sandbox hosts file ".../hosts": open /etc/hosts: no such file or directory
```

the symptom is a Pod stuck in `ContainerCreating` with repeated `FailedCreatePodSandBox` warnings. first seen on `monitoring/kube-prometheus-stack-prometheus-node-exporter-*`, which reaches redqueen because that DaemonSet tolerates every taint.

fix as root on redqueen:

```sh
sudo tee /etc/hosts >/dev/null <<'EOF'
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
EOF
sudo chown root:root /etc/hosts
sudo chmod 0644 /etc/hosts
```

do **not** add a `127.0.1.1 <hostname>` line. `files` precedes `myhostname` and `resolve`, so such an entry would shadow the host's real address and change what `getent hosts amd-halo` returns. `/` is btrfs read-write, so no remount is needed, but `/etc/hostname` ships mode `0444`, so treat `/etc` as image-managed and re-check this file after any OS update or reimage.

kubelet retries sandbox creation every ~12 seconds and containerd re-reads the host file per sandbox, so the pending Pod recovers without restarting `k3s-agent`.

## 6. rollback

```sh
sudo /usr/local/bin/k3s-agent-uninstall.sh
sudo rm -f /root/k3s-agent-token
```

```sh
kubectl delete node redqueen
```

this removes the agent's local state and the Node object; it does not touch `/srv/ai`, the inference services, or the host firewall.

## notes

- the agent cannot schedule anything without the taint toleration, so the join is inert until the LiteLLM placement change is separately approved.
- **`git pull` on redqueen deploys.** `link-runtime.sh` installs `post-merge` and `post-rewrite` hooks into `core.hooksPath` (`/srv/ai/model-router/git-hooks`), and they exec `.githooks/redqueen-pull-upgrade`, which runs the drain-and-deploy flow. a docs-only pull still revalidates runtime manifests and reapplies user units. observed 2026-11-02 while fast-forwarding this file: the pull reported `deployed <rev>; APIs healthy`. use `git fetch` plus inspection when you do not intend to deploy.
- `--node-label` and `--node-taint` are registration-time only. change them later with `kubectl`, not by editing the k3s unit.
- the node name is pinned to `redqueen`; the host's own hostname (`amd-halo`) and DNS alias do not change.
- LiteLLM's 1 CPU / 4 GiB request fits the remaining allocatable memory and CPU after the reservations above.
- keep the raw inference ports loopback or firewall-restricted as they are; the join does not add a public inference listener.
- kubelet binds TCP `10250` on all interfaces, including IPv6. the tracked `hosts/redqueen/nftables.d/redqueen-inference.nft` now restricts it to the same `clients_v4` set as the inference ports and drops it for IPv6. kubelet currently rejects anonymous requests (`/healthz`, `/metrics`, and `/pods` all return `401`), so this is hardening rather than an open data leak.
- `link-system-runtime.sh` uses `install_exact_copy`, which **refuses to overwrite** a root-owned file that differs from the tracked source. after editing the tracked `.nft`, apply it explicitly and reload; a plain linker run will report "root-owned file differs from source; preserving it".

  ```sh
  sudo nft -c -f /srv/ai/inference/hosts/redqueen/nftables.d/redqueen-inference.nft
  sudo install -o root -g root -m 0644 \
    /srv/ai/inference/hosts/redqueen/nftables.d/redqueen-inference.nft \
    /etc/nftables.d/redqueen-inference.nft
  sudo systemctl reload redqueen-inference-firewall
  sudo nft list table inet redqueen_inference
  ```

  then re-check from an unlisted host that `10250` is closed and `8081`/`8190`/`8191` still behave as before.
- `/etc/nftables.d/` contains two stale backup entries from 2026-10-10: a real `redqueen-inference.nft.backup.20261010140213` and a *symlink* named `redqueen-inference.nft.backup.20261010140304` that points into this checkout. nothing globs that directory, so they are inert, but they are not part of the tracked configuration and should be reviewed by the owner before cleanup.
