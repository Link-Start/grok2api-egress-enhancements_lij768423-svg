# Grok2API egress quality guard

**Primary path: [lij768423-svg/grok2api](https://github.com/lij768423-svg/grok2api) + Quality Guard sidecar.** The CPA plugin is optional, not the default.

## One-shot install prompt

Paste this to your AI, then append every residential sticky:

```text
Follow this doc exactly:
https://github.com/lij768423-svg/grok2api-egress-enhancements/blob/main/docs/AI_GROK2API_INSTALL.md

Primary path: lij768423-svg/grok2api + Quality Guard sidecar. Do not install CPA.
Use every residential sticky: one Mihomo listener + one Grok2API node per session.
A single res-01 node, or merging many stickies into one pool, is not done.

Machine: Linux + Docker, install into ~/grok-stack (new directory).
Email is optional. Bring the exits and Guard up first.

Residential lines (one per line):

```

Chinese walkthrough: [docs/AI_GROK2API_INSTALL.md](./docs/AI_GROK2API_INSTALL.md). Generator: [`scripts/from_residential.py`](./scripts/from_residential.py).

This is an unofficial enhancement distribution for [chenyme/grok2api](https://github.com/chenyme/grok2api): immediate fixed-proxy recovery, egress quality-guard patches, and the production Quality Guard sidecar (`/quality-guard`). The repository does not copy the complete upstream source.

Current baseline:

- Upstream release: `v3.1.4` / `909bb810` (0006–0014 / #956–#959 #966–#968 are already upstream)
- Today's delta: TUI hold evidence + serde required fields + empty-stream retry (`patches/0015-*.patch` … `0020-*.patch`, PRs #977–#981 #984)
- Runnable fork: [lij768423-svg/grok2api](https://github.com/lij768423-svg/grok2api) `main`

If you are still on `v3.0.11`, keep using the legacy patch `patches/0001-feat-add-egress-recovery-and-quality-guard.patch` (closed [#837](https://github.com/chenyme/grok2api/pull/837)).

## Features

### Request-path missing-thinking hold (v3.1.4+)

- Buffer thinking-model SSE until **streamed** thinking appears (reasoning/summary deltas or `encrypted_content`). `usage.reasoning_tokens`, empty reasoning items, and the Chat SSE stub do **not** count.
- Missing thinking (`output ≥ 32`, no streamed thinking) is **not delivered**. Another account is tried.
- Grok TUI always sends a `tools` schema, and the next turn after local tools already contains `function_call_output`. Both still hold.
- Empty `response.completed` / `[DONE]` fails immediately as `upstream_stream_empty` instead of waiting for idle under HTTP 200.
- Abort trailers emit `response.failed` with required `model`; `output_text.annotations` is filled as `[]`. TUI 0.2.93 treats `response.incomplete` as fatal truncation.
- Up to 6 attempts (first + 5 switches). All misses → `503 quality_degraded`.
- First miss: 24h scheduling cooldown (`accountCooldown`). A later miss after that cooldown disables the account.
- Empty-stream penalty is `idleAccountCooldown`; `POST /api/admin/v1/accounts/:id/clear-cooldown` clears it.
- Grok TUI compaction still sets `skipQualityHold`. Do not inject `compaction_trigger`.
- Off by default: `qualityGuard.requestRetry.enabled: false`.

### Immediate fixed-proxy recovery

- A pre-submission transport failure persists cooldown state and starts an immediate background probe.
- Concurrent failures for one node share a single probe.
- A later bound request waits for at most five seconds, reloads persisted state after healthy recovery, and continues early.
- Request cancellation stops the wait without canceling the shared probe.
- Submitted generation requests, authentication failures, quota exhaustion, and rate limits are never replayed by this mechanism.
- Upstream's existing proxy-pool mode keeps fresh-tunnel isolation, so one rotating exit failure does not cool the whole pool.

### Egress quality guard

- Passive audits use the grok2api panel formula `output tokens / (duration - first token)`; output tokens include reasoning tokens.
- **Passive hard-threshold hits quarantine immediately**. Soft hits still trigger a fixed-prompt active confirmation and require consecutive strikes.
- Active soft and hard thresholds, consecutive probe-error handling, minimum healthy-node protection, quarantine, and recovery.
- Fail-closed quarantine before confirmation, with same-IP confirmation for short buffered bursts to avoid false rotation.
- A trusted per-node rotation webhook and a 1024Proxy `sid-...-t-...` sticky-session rotator.
- One real-model check per new IP: healthy results restore immediately; anomalous or indeterminate results remain isolated.
- Account-selection failures are deferred without counting a proxy error or rotating the IP.
- If a target node's bound accounts are unavailable, administrator probes borrow any healthy Build account while still forcing the physical request through the node under test. Ordinary traffic is unchanged.
- If the entire account pool is unavailable, the guard uses a separate long backoff and suppresses duplicate no-account logs while keeping the node isolated.
- Admin UI, manual diagnostics, hot-reloadable policy, and persistent statistics.
- One replaceable toast per node action, with manual tests disabled while a node is quarantined or rotating.
- Create, edit, delete, enable, disable, and refresh Build proxy nodes directly from the node-quality table.
- Select individual or all nodes and batch enable, disable, or delete them with destructive-action confirmation.
- Automatically discover proxied Build nodes when `QUALITY_GUARD_NODE_IDS` is empty while publishing resolved IDs for compatibility with older admin pages.
- Python sidecar, Docker Compose and systemd examples, security notes, and bilingual documentation.

### Degraded-account monitor (v3.1.2 delta)

- Adds a Quality Guard tab that classifies user streaming requests (excluding quality-test probes) as `buffered_burst` / `soft_tps` / `hard_tps`.
- Same panel formula: `outputTokens * 1000 / (durationMs - firstTokenMs)`, defaults soft 500 / hard 1000; windows shorter than 1s that reach soft are `buffered_burst`.
- Windows: 1h / 6h / 24h / 7d. Filter by email/ID, schedule status, class, and hit count.
- Timeline bars grow from the bottom. Any row is selectable; bulk mute or unmute uses the existing account batch API with string `ids`.
- Endpoint: `GET /api/admin/v1/request-audits/degrade-accounts`.

### CPA-native egress guard plugin

`cpa-plugin/` is now the **v1.1.0 pure-CPA plugin** (same features as 1.0.9; this docs drop ships grok2api patches 0015–0020). It has no runtime dependency on Grok2API: it uses CPA Host APIs for auth files and usage events, binds `proxy_url` stickily to egress nodes, and provides node CRUD, line-based bulk import, batch operations, connectivity/real-model tests, configurable probe profiles (throughput / expected-marker / custom prompt), quarantine migration, hot-reloadable policy, statistics, events, and light/dark themes. In v1.0.9, active probes can verify a last-line or regex marker. In v1.0.8, store-install registration no longer blocks on a full auth scan (fixes plugins stuck as inactive/unregistered with many accounts). In v1.0.7, CPA scheduling skips quarantined or cooling egresses; credential, quota, and permission failures are recorded as ignored instead of quarantining a node; migrations are read-back verified; and an optional allowlisted internal IP-rotation webhook is available. See [cpa-plugin/README.md](./cpa-plugin/README.md) for build instructions and the Chinese [AI deployment and operations guide](./cpa-plugin/AI_USAGE_GUIDE.md) for proxy topology, capacity planning, quarantine recovery, and forced residential-IP rotation.

For the recommended end-to-end topology (residential/Resin -> Mihomo sharding and listeners -> Grok2API/CPA egress nodes -> Quality Guard detection, drain, rotation, and re-probing), see [Recommended egress deployment](./docs/RECOMMENDED_DEPLOYMENT.md).

CPA itself does not degrade model quality. This optional plugin acts as an egress circuit breaker for multi-account, multi-egress deployments; single-account or stable static-proxy installations may not need it.

The quality guard is a heuristic circuit breaker, not proof that upstream model capability changed. Immediate hard quarantine is intentionally aggressive; raise `hard_tps` when false positives are more costly. Soft anomalies still require confirmation probes.


## Apply directly

From a clean grok2api checkout:

```sh
git fetch --tags origin
git checkout -b tui-hold-serde v3.1.4
git am --3way /path/to/grok2api-egress-enhancements/patches/0015-fix-quality-thinking-evidence.patch
git am --3way /path/to/grok2api-egress-enhancements/patches/0016-fix-responses-annotations.patch
git am --3way /path/to/grok2api-egress-enhancements/patches/0017-fix-hold-tui-tool-turns.patch
git am --3way /path/to/grok2api-egress-enhancements/patches/0018-fix-empty-completed-retry.patch
git am --3way /path/to/grok2api-egress-enhancements/patches/0019-fix-clear-account-cooldown.patch
git am --3way /path/to/grok2api-egress-enhancements/patches/0020-fix-responses-abort-failed-model.patch
```

Easier: clone [lij768423-svg/grok2api](https://github.com/lij768423-svg/grok2api) `main`. Do not `git am` 0006–0014 onto v3.1.4. On `v3.0.11`, apply `patches/0001-feat-add-egress-recovery-and-quality-guard.patch` instead. For newer upstream versions, follow [AI_MERGE_GUIDE.md](./docs/AI_MERGE_GUIDE.md).

## Validate

```sh
go test ./...
python3 -m unittest -v \
  tools/egress-quality-guard/quality_guard_test.py \
  tools/egress-quality-guard/session_rotator_test.py  # 26 tests
cd frontend
pnpm lint
pnpm build
```

## Security and privacy

Never provide real environment files, application config, databases, state volumes, proxy URLs, account credentials, or production logs to an AI merge tool. The upstream source, this patch, and sanitized test failures are sufficient.

## Related projects

- [Grok Register + Live Panel](https://github.com/lij768423-svg/grok-register-panel): a separate Camoufox-based Grok registration workflow and web control panel with multiple email backends, an external proxy pool, egress checks, an ASN blacklist, runtime statistics, and account recovery. It is not bundled with this patch.

## Friends

- [LINUX DO](https://linux.do) — A new kind of community

## License and attribution

The patch is distributed under the upstream MIT license. Preserve the upstream LICENSE, copyright notices, and Git history. This repository is not an official grok2api release and does not imply upstream endorsement.
