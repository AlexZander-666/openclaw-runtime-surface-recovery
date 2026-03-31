---
name: openclaw-runtime-surface-recovery
description: Use this skill when OpenClaw receives inbound channel messages but does not send replies, especially when logs mention missing bundled plugin public surfaces such as speech-core/runtime-api.js or image-generation-core/runtime-api.js. It covers runtime-source verification, systemd user service inspection, channel status checks, log-driven root cause analysis, repo-side fixes for premature runtime loading, and operational recovery via OPENCLAW_BUNDLED_PLUGINS_DIR overrides.
---

# OpenClaw Runtime Surface Recovery

## Overview

Use this skill for OpenClaw reply failures where inbound delivery works but outbound replies never send, or when background lanes fail with errors like `Unable to resolve bundled plugin public surface .../runtime-api.js`.

Focus on two separate classes of failure:

1. Runtime packaging/loader failure: the active installed build cannot resolve bundled extension public surfaces.
2. Code-path bug: OpenClaw loads a runtime surface even when the feature should be off, such as TTS loading while `tts=off`.

## Workflow

### 1. Confirm what is actually running

Do not assume the machine is running the checked-out repo.

Start with:

```bash
which openclaw
readlink -f "$(which openclaw)"
openclaw --version
ps -ef | rg 'openclaw-gateway|openclaw\.mjs gateway run'
ss -ltnp | rg 18789
```

If the gateway is systemd-managed, inspect the service:

```bash
systemctl --user status openclaw-gateway.service --no-pager
systemctl --user show openclaw-gateway.service -p ExecStart -p Environment -p FragmentPath -p DropInPaths
sed -n '1,220p' ~/.config/systemd/user/openclaw-gateway.service
```

Key question: is the machine running:

- a repo build from `dist/`
- a repo dev entry
- a global install such as `/usr/lib/node_modules/openclaw/dist/index.js`

### 2. Verify the channel is alive

Check the channel before changing code:

```bash
openclaw channels status --probe
openclaw plugins list
```

For Weixin specifically, verify account state under `~/.openclaw/openclaw-weixin/` and confirm inbound sync files are fresh.

If status says the channel is `running` and inbound artifacts are moving, the failure is probably in reply generation or reply delivery, not login.

### 3. Read the active log, not an old one

Open the latest OpenClaw log under `/tmp/openclaw/`:

```bash
ls -lt /tmp/openclaw | head
tail -n 200 /tmp/openclaw/openclaw-YYYY-MM-DD.log
rg -n 'dispatchReplyFromConfig|runtime-api.js|getUpdates error|Embedded agent failed before reply' /tmp/openclaw/openclaw-YYYY-MM-DD.log
```

Interpretation:

- `dispatchReplyFromConfig: error ... speech-core/runtime-api.js`
  The reply pipeline crashed before channel send.
- `Embedded agent failed before reply ... image-generation-core/runtime-api.js`
  A background or agent lane is loading a missing bundled public surface.
- channel inbound logs without corresponding send logs
  The channel likely receives messages correctly; the crash is earlier.

### 4. Check whether the feature should even be loading

Before assuming packaging is the only bug, verify whether the runtime feature is logically enabled.

For TTS:

- inspect session store for `ttsAuto`
- inspect resolved config
- inspect resolved TTS status snapshot

Useful files and checks:

```bash
rg -n 'ttsAuto|openclaw-weixin' ~/.openclaw/agents/main/sessions/sessions.json
```

If TTS resolves to `off` but `dispatch-from-config` still imports TTS runtime, that is a code bug and should be fixed in repo.

## Known Failure Pattern

### Symptom

- inbound Weixin message arrives
- OpenClaw does not reply
- logs show:

```text
Unable to resolve bundled plugin public surface speech-core/runtime-api.js
```

### Root cause split

1. Repo-side logic bug:
   `dispatch-from-config` eagerly loads TTS runtime before checking whether TTS is actually enabled.

2. Operational packaging issue:
   the running global install does not contain certain bundled extension public surfaces in `dist/extensions`, so any eager load of those surfaces crashes.

## Repo Fix Pattern

When a runtime feature is optional, do not import its runtime surface until after a lightweight enabled-state check.

For the TTS case:

- compute a cheap status snapshot first
- skip runtime import when snapshot resolves to off
- keep final/block/tool delivery payloads unchanged when TTS is disabled

Validation target:

- add a regression test that proves `maybeApplyTtsToPayload` is never called when session or resolved status says `off`
- keep existing routing behavior intact

Example verification commands:

```bash
pnpm test -- src/auto-reply/reply/dispatch-from-config.test.ts src/auto-reply/reply/route-reply.test.ts
pnpm tsgo
pnpm build
```

## Operational Recovery

If the running service is a global install and missing bundled public surfaces, use a systemd user drop-in to point bundled plugin resolution at the repo source tree:

```ini
[Service]
Environment=OPENCLAW_BUNDLED_PLUGINS_DIR=/path/to/OpenClaw/extensions
```

Typical location:

`~/.config/systemd/user/openclaw-gateway.service.d/override.conf`

Then reload and restart:

```bash
systemctl --user daemon-reload
systemctl --user restart openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager
```

After restart, verify:

```bash
systemctl --user show openclaw-gateway.service -p Environment -p ExecMainPID
ss -ltnp | rg 18789
openclaw channels status --probe
rg -n 'speech-core/runtime-api.js|image-generation-core/runtime-api.js|dispatchReplyFromConfig: error' /tmp/openclaw/openclaw-YYYY-MM-DD.log
```

Expected outcome:

- `OPENCLAW_BUNDLED_PLUGINS_DIR` appears in the service environment
- gateway listens on 18789
- channel status becomes reachable and running
- new logs stop showing bundled public surface resolution errors

## Decision Rules

- If the active process is still the old global install, repo fixes alone will not change live behavior.
- If logs stop showing surface-resolution errors after the service override, operational recovery succeeded even if the installed package is still old.
- If feature state is `off` and the runtime still loads, fix the repo logic anyway; the env override is only a runtime workaround.
- If both repo fix and service override are needed, land the repo patch and apply the override separately.

## Deliverables

When using this skill, end with:

1. active runtime source
2. exact failing log signature
3. code bug vs operational bug split
4. repo files changed, if any
5. service or environment changes applied on the machine
6. verification commands and results
