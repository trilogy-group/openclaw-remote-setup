---
name: openclaw-prereqs
description: Install bootstrap prerequisites (Node 24, beads, jq) on a fresh Ubuntu 24.04 box and init
ialize the shared beads task graph that openclaw-harden and openclaw-install will track their work in.
 Idempotent. Run this first on any new box. Skill 1 of 3.
disable-model-invocation: true
---

# OpenClaw bootstrap, phase 1: prerequisites

This is skill 1 of 3. It installs the foundational tooling and the shared progress-tracking infrastruc
ture that the next two skills use.

Order:
1. `/openclaw-prereqs` (this skill)
2. `/openclaw-harden`
3. `/openclaw-install`

Each later skill checks for the prior phase's completion marker in beads and refuses to start if it is
n't closed.

## Why beads, and why first

Models systematically skip steps in long checklists because partially-complete tasks sound plausible. 
Beads makes skipping structurally impossible: every step is a graph node, blocked-by its predecessors,
 and unable to close without a recorded validation reason. We install beads before anything else so th
e harden and install skills inherit this discipline from the first command they run.

The third reading-list article, [How to Fix Your AI Agents That Keep Cutting Corners](https://trilogya
i.substack.com/p/how-to-fix-your-ai-agents-keep-cutting), is the long form of that argument.

## Preflight

If any check fails, stop and surface the problem.

```bash
# 1. OS
. /etc/os-release
[ "$VERSION_ID" = "24.04" ] || { echo "STOP: targets Ubuntu 24.04, found $PRETTY_NAME"; exit 1; }

# 2. Privilege
if [ "$(id -u)" -ne 0 ] && ! sudo -n true 2>/dev/null; then
  echo "STOP: need root or passwordless sudo"; exit 1
fi

# 3. Network
curl -fsS https://registry.npmjs.org/ -o /dev/null -w "npm reachable: %{http_code}\n" || { echo "STOP:
 npm registry unreachable"; exit 1; }

# 4. Disk (need 10 GB free)
df -BG --output=avail / | tail -1
```

## Step 1: Install Node 24, jq, build-essential

```bash
if ! command -v node >/dev/null 2>&1 || ! node --version | grep -Eq '^v(24|22\.1[4-9]|22\.[2-9][0-9])'
; then
  curl -fsSL https://deb.nodesource.com/setup_24.x | bash -
fi
apt update && apt install -y nodejs build-essential jq curl
node --version
jq --version
```

Validate the Node version is `v24.*` or `v22.14+`. Anything older means an old NodeSource cache; force
 a rerun of the setup script.

## Step 2: Install beads

```bash
command -v bd >/dev/null 2>&1 || npm install -g @beads/bd
bd --version
```

If `bd --version` fails after install, surface the npm error. Common fix: `npm install -g npm@latest` 
then retry.

## Step 3: Initialize the bootstrap beads project

```bash
mkdir -p /root/.claude-bootstrap
cd /root/.claude-bootstrap
[ -d .beads ] || bd init
ls -la .beads | head -5
```

The bootstrap beads database lives at `/root/.claude-bootstrap/.beads`. The runtime beads database tha
t OpenClaw agents will use (set up by `/openclaw-install` step 6) lives at `/home/<user>/.openclaw/wor
kspace/.beads`. They are separate by design and do not interfere.

## Step 4: Initialize the shared inputs file

The inputs file is where the three skills persist parameters (target username, timezone, API keys). We
 create it now with mode 600 so later writes inherit the restriction.

```bash
INPUTS=/root/.openclaw-bootstrap-inputs.json
[ -f "$INPUTS" ] || echo '{}' > "$INPUTS"
chmod 600 "$INPUTS"
ls -la "$INPUTS"
```

## Step 5: Discover beads CLI flags

Beads CLI shape evolves. Run these and read the help output before issuing any `bd issue create` or `b
d issue update` command. Use the actual flag names from your output, not the placeholders below.

```bash
bd --help
bd issue --help 2>&1 | head -40
bd issue create --help 2>&1 | head -40
bd issue update --help 2>&1 | head -40
bd issue list --help 2>&1 | head -20
bd ready --help 2>&1 | head -20 || true
```

Note the exact flag names for: `--title`, `--type`, `--status`, `--close-reason` (or `--reason`), `--b
locked-by`, and how the JSON output is shaped (so `jq -r '.id'` works). Adapt commands in the harden a
nd install skills accordingly.

## Step 6: Create the prereqs-complete marker

This issue is the signal the next skill (`/openclaw-harden`) checks for. Create it idempotently.

```bash
cd /root/.claude-bootstrap

EXISTING=$(bd issue list --json 2>/dev/null | jq -r '.[] | select(.title == "prereqs-complete") | .id'
 | head -1)

if [ -z "$EXISTING" ]; then
  MARKER_ID=$(bd issue create --title "prereqs-complete" --type task --json | jq -r '.id')
  bd issue update "$MARKER_ID" --status closed --close-reason "Node $(node --version), beads $(bd --ve
rsion), inputs file at $INPUTS, beads dir at /root/.claude-bootstrap/.beads"
fi

bd issue list --status closed --json 2>/dev/null | jq -r '.[] | select(.title == "prereqs-complete") |
 "Marker: \(.id) - \(.title) - status=\(.status)"'
```

If the marker existed before this skill ran (idempotent re-invocation), confirm its close-reason is st
ill accurate and update if not.

## Done

Phase 1 complete. Run `/openclaw-harden` next.

Expected state after this skill:
- `node --version` returns v24.x (or v22.14+)
- `bd --version` returns a version
- `/root/.claude-bootstrap/.beads/` exists
- `/root/.openclaw-bootstrap-inputs.json` exists with mode 600
- `bd issue list` shows a closed `prereqs-complete` issue

If any of those are missing, this skill did not complete. Re-run it.