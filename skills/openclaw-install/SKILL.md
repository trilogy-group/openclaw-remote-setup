---
name: openclaw-install
description: Install OpenClaw on a hardened Ubuntu 24.04 box. Installs Claude Code + cc-openclaw skills, OpenClaw gateway as a systemd user service, secrets via ${ENV_VAR} placeholders, Telegram bot (with privacy-mode gotcha), Xvfb + Chrome headless browser stack, runtime beads at .openclaw/workspace/.beads, optional GNU Stow dotfiles, SSH tunnel instructions, first cron job, and a final audit. Tracked in beads. Skill 3 of 3. Requires /openclaw-harden to have run first.
disable-model-invocation: true
---

# OpenClaw bootstrap, phase 3: install

This is skill 3 of 3. It installs OpenClaw and adjacent services on top of a hardened box.

Order:
1. `/openclaw-prereqs` (prerequisite)
2. `/openclaw-harden` (prerequisite)
3. `/openclaw-install` (this skill)

## Preflight: confirm phases 1 and 2 ran

```bash
INPUTS=/root/.openclaw-bootstrap-inputs.json
BEADS_HOME=/root/.claude-bootstrap

[ -d "$BEADS_HOME/.beads" ] || { echo "STOP: bootstrap beads not initialized. Run /openclaw-prereqs fi
rst."; exit 1; }
[ -f "$INPUTS" ] || { echo "STOP: inputs file missing. Run /openclaw-prereqs first."; exit 1; }
command -v bd >/dev/null 2>&1 || { echo "STOP: beads CLI missing."; exit 1; }
command -v jq >/dev/null 2>&1 || { echo "STOP: jq missing."; exit 1; }

cd "$BEADS_HOME"
P=$(bd issue list --status closed --json 2>/dev/null | jq -r '.[] | select(.title == "prereqs-complete
") | .id' | head -1)
H=$(bd issue list --status closed --json 2>/dev/null | jq -r '.[] | select(.title == "harden-complete"
) | .id' | head -1)
[ -n "$P" ] || { echo "STOP: prereqs-complete marker not closed. Run /openclaw-prereqs."; exit 1; }
[ -n "$H" ] || { echo "STOP: harden-complete marker not closed. Run /openclaw-harden."; exit 1; }

TARGET_USER=$(jq -r '.target_username // empty' "$INPUTS")
[ -n "$TARGET_USER" ] || { echo "STOP: target_username not in inputs. /openclaw-harden should have set
 it."; exit 1; }
id "$TARGET_USER" >/dev/null || { echo "STOP: user $TARGET_USER does not exist. /openclaw-harden shoul
d have created it."; exit 1; }

echo "Preflight OK. Markers: prereqs=$P, harden=$H. Target user: $TARGET_USER."
```

## Critical safety rules

1. **Never `git add` or `git commit` anything that contains a secret.** Grep staged diffs for `sk-ant-
`, `AAG`, `GOCSPX-`, `ghp_`, `xoxb-`, `AKIA`. Abort if found.
2. **Never bind the gateway to `0.0.0.0`.** Loopback only. The SSH tunnel is the access path.
3. **Never echo a secret after capture.** Show only `prefix...suffix` (first 4, last 4).
4. **If a step's validation fails, stop.** Set the beads issue to `blocked` and surface the error.
5. **Do not close a beads issue without recording validation evidence in the close reason.**
6. **Do not auto-push to a git remote.** Always ask the user explicitly.

## Inputs to collect

| Input | When | Notes |
|---|---|---|
| anthropic_api_key | I3 | Paste via AskUserQuestion. Validate format `sk-ant-*`. |
| telegram_bot_token | I4 | From BotFather. |
| dotfiles_repo_url | I7 (optional) | SSH URL like `git@github.com:user/repo.git`. |

Persist to `$INPUTS`. Never echo back.

## Step graph: create or reuse

```bash
cd /root/.claude-bootstrap
MAP=/root/.claude-bootstrap/install-ids.env

ensure_issue() {
  local var="$1"; local title="$2"; shift 2
  local existing
  existing=$(bd issue list --json 2>/dev/null | jq -r --arg t "$title" '.[] | select(.title == $t) | .
id' | head -1)
  if [ -n "$existing" ]; then
    printf -v "$var" '%s' "$existing"
  else
    local args=()
    for blocker in "$@"; do args+=(--blocked-by "${!blocker}"); done
    local id
    id=$(bd issue create --title "$title" --type task "${args[@]}" --json | jq -r '.id')
    printf -v "$var" '%s' "$id"
  fi
  printf '%s=%s\n' "$var" "${!var}" >> "$MAP"
  echo "$var=${!var} ($title)"
}

HARDEN_MARKER=$(bd issue list --status closed --json | jq -r '.[] | select(.title == "harden-complete"
) | .id' | head -1)

ensure_issue I_PHASE "install-phase"
ensure_issue I1 "Install 1: Claude Code and cc-openclaw skills" HARDEN_MARKER
ensure_issue I2 "Install 2: OpenClaw and onboard daemon" I1
ensure_issue I3 "Install 3: secrets via env-var placeholders" I2
ensure_issue I4 "Install 4: Telegram bot and privacy gotcha" I3
ensure_issue I5 "Install 5: headless browser stack" I3
ensure_issue I6 "Install 6: runtime beads at .openclaw/workspace/.beads" I3
ensure_issue I7 "Install 7: dotfiles with GNU Stow (optional)" I3 I4 I5 I6
ensure_issue I8 "Install 8: SSH tunnel from laptop (informational)" I7
ensure_issue I9 "Install 9: first cron job" I7
ensure_issue I10 "Install 10: final audit" I9
bd issue update "$I_PHASE" --blocked-by "$I1" --blocked-by "$I2" --blocked-by "$I3" --blocked-by "$I4"
 \
    --blocked-by "$I5" --blocked-by "$I6" --blocked-by "$I7" --blocked-by "$I8" --blocked-by "$I9" --b
locked-by "$I10"

bd ready
```

## Tracking pattern

For each step: `bd ready` → `--status in_progress` → run → validate → `--status closed --close-reason 
"<output>"`. On failure: `--status blocked --reason "<error>"`, stop.

---

## I1: Claude Code and cc-openclaw skills

```bash
bd issue update "$I1" --status in_progress

# Claude Code is likely already installed (we are running from it), but ensure it.
which claude || npm install -g @anthropic-ai/claude-code
apt install -y stow git

TARGET_USER=$(jq -r '.target_username' "$INPUTS")
sudo -u "$TARGET_USER" -i bash <<'EOSU'
[ -d ~/cc-openclaw ] || git clone https://github.com/openclaw/cc-openclaw ~/cc-openclaw
cd ~/cc-openclaw && stow -t ~ .
EOSU

OUT=$(ls "/home/$TARGET_USER/.claude/skills/" 2>/dev/null | head)
[ -n "$OUT" ] || { bd issue update "$I1" --status blocked --reason "cc-openclaw skills not visible aft
er stow"; exit 1; }
bd issue update "$I1" --status closed --close-reason "claude=$(which claude); skills:$(echo "$OUT" | w
c -l) entries"
```

## I2: OpenClaw and onboard daemon

```bash
bd issue update "$I2" --status in_progress

TARGET_USER=$(jq -r '.target_username' "$INPUTS")
npm install -g openclaw@latest
sudo -u "$TARGET_USER" -i bash -c 'openclaw --version'
```

Now run `openclaw onboard --install-daemon` AS THE TARGET USER. The onboard is interactive: surface it
s prompts verbatim to the human. When asked where to bind, ensure they choose `127.0.0.1:18789`. If th
e onboard tries to bind to `0.0.0.0`, intercept and insist on loopback.

```bash
sudo -u "$TARGET_USER" -i bash -c 'openclaw onboard --install-daemon'
```

Validate:

```bash
OUT=$(sudo -u "$TARGET_USER" -i systemctl --user status openclaw-gateway --no-pager | head -10)
echo "$OUT" | grep -q 'active (running)' || { bd issue update "$I2" --status blocked --reason "gateway
 not active: $OUT"; exit 1; }
bd issue update "$I2" --status closed --close-reason "$OUT"
```

## I3: Secrets via env-var placeholders

`AskUserQuestion` for the Anthropic API key if not already in `$INPUTS`. Validate prefix `sk-ant-`. Ne
ver echo it back; show `sk-ant-...<last-4>` if confirming.

```bash
bd issue update "$I3" --status in_progress

TARGET_USER=$(jq -r '.target_username' "$INPUTS")
ANTHROPIC_KEY=$(jq -r '.anthropic_api_key // empty' "$INPUTS")
GATEWAY_TOKEN=$(openssl rand -hex 24)

sudo -u "$TARGET_USER" -i bash <<EOSU
touch ~/.openclaw/.env
chmod 600 ~/.openclaw/.env
cat > ~/.openclaw/.env <<EOF
ANTHROPIC_API_KEY=${ANTHROPIC_KEY:-CHANGE_ME}
TELEGRAM_BOT_TOKEN=CHANGE_ME_AFTER_I4
OPENCLAW_GATEWAY_TOKEN=$GATEWAY_TOKEN
DISPLAY=:1
EOF
EOSU

jq --arg t "$GATEWAY_TOKEN" '.openclaw_gateway_token = $t' "$INPUTS" > "$INPUTS.tmp" && mv "$INPUTS.tm
p" "$INPUTS"

# Wire EnvironmentFile into the systemd unit
sudo -u "$TARGET_USER" -i bash <<'EOSU'
UNIT=~/.config/systemd/user/openclaw-gateway.service
grep -q 'EnvironmentFile=' "$UNIT" || sed -i '/^\[Service\]/a EnvironmentFile=%h/.openclaw/.env' "$UNI
T"
systemctl --user daemon-reload
systemctl --user restart openclaw-gateway
EOSU

# Surgically rewrite openclaw.json placeholders
sudo -u "$TARGET_USER" -i bash <<'EOSU'
CFG=~/.openclaw/openclaw.json
TMP=$(mktemp)
jq '
  .gateway.auth.token = "${OPENCLAW_GATEWAY_TOKEN}"
  | .channels.telegram.accounts.default.token = "${TELEGRAM_BOT_TOKEN}"
  | .models.anthropic.apiKey = "${ANTHROPIC_API_KEY}"
' "$CFG" > "$TMP" && mv "$TMP" "$CFG"
systemctl --user restart openclaw-gateway
EOSU

OUT=$(sudo -u "$TARGET_USER" -i bash -c 'systemctl --user is-active openclaw-gateway')
[ "$OUT" = "active" ] || { bd issue update "$I3" --status blocked --reason "gateway not active after e
nv wiring"; exit 1; }
bd issue update "$I3" --status closed --close-reason ".env created chmod 600; openclaw.json placeholde
rs wired; gateway active"
```

## I4: Telegram bot and the privacy gotcha

Walk the user through BotFather `/newbot`. Capture token via `AskUserQuestion`, persist to `$INPUTS`, 
write into `.env`:

```bash
bd issue update "$I4" --status in_progress

TARGET_USER=$(jq -r '.target_username' "$INPUTS")
TOKEN=$(jq -r '.telegram_bot_token' "$INPUTS")
sudo -u "$TARGET_USER" -i bash -c "sed -i 's|^TELEGRAM_BOT_TOKEN=.*|TELEGRAM_BOT_TOKEN=$TOKEN|' ~/.ope
nclaw/.env && systemctl --user restart openclaw-gateway"
```

Walk the user through:
1. Create a Telegram group.
2. Add the bot.
3. Send a tagged message.

If the bot does not respond, surface the privacy-mode gotcha:

> Telegram bots default to privacy mode in groups, meaning they only see messages mentioning them by h
andle. Switching it off via BotFather (`/setprivacy` → Disable) does NOT apply retroactively. Easiest 
fix: promote the bot to admin in the group. Alternative: remove and re-add.

Block via `AskUserQuestion` until the bot responds. Persist the resolution into `$INPUTS.telegram_priv
acy_resolution`. Then:

```bash
bd issue update "$I4" --status closed --close-reason "Telegram bot responding; resolution: $(jq -r '.t
elegram_privacy_resolution // "first-try"' "$INPUTS")"
```

## I5: Headless browser stack

```bash
bd issue update "$I5" --status in_progress

apt install -y xvfb tigervnc-standalone-server novnc websockify
wget -qO - https://dl.google.com/linux/linux_signing_key.pub | gpg --dearmor -o /usr/share/keyrings/go
ogle.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/google.gpg] http://dl.google.com/linux/chrome/deb/
 stable main" > /etc/apt/sources.list.d/google-chrome.list
apt update && apt install -y google-chrome-stable

TARGET_USER=$(jq -r '.target_username' "$INPUTS")
sudo -u "$TARGET_USER" -i bash <<'EOSU'
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/xvfb.service <<'EOF'
[Unit]
Description=Xvfb on display :1
After=default.target

[Service]
ExecStart=/usr/bin/Xvfb :1 -screen 0 1920x1080x24
Restart=always

[Install]
WantedBy=default.target
EOF
systemctl --user daemon-reload
systemctl --user enable --now xvfb
EOSU

OUT=$(sudo -u "$TARGET_USER" -i bash -c 'systemctl --user is-active xvfb')
[ "$OUT" = "active" ] || { bd issue update "$I5" --status blocked --reason "xvfb not active"; exit 1; 
}
bd issue update "$I5" --status closed --close-reason "google-chrome-stable installed; xvfb active on :
1"
```

## I6: Runtime beads at .openclaw/workspace/.beads

Sets up a SECOND beads database, separate from the bootstrap one. The bootstrap beads at `/root/.claud
e-bootstrap/.beads` tracks THIS skill's progress. The runtime beads at `/home/<user>/.openclaw/workspa
ce/.beads` is what OpenClaw agents will use for their own task graphs.

```bash
bd issue update "$I6" --status in_progress

TARGET_USER=$(jq -r '.target_username' "$INPUTS")
sudo -u "$TARGET_USER" -i bash <<'EOSU'
mkdir -p ~/.openclaw/workspace
cd ~/.openclaw/workspace
[ -d .beads ] || bd init
grep -q '^BEADS_DIR=' ~/.openclaw/.env || echo "BEADS_DIR=$HOME/.openclaw/workspace/.beads" >> ~/.open
claw/.env
systemctl --user restart openclaw-gateway
EOSU

OUT=$(sudo -u "$TARGET_USER" -i bash -c 'ls -la ~/.openclaw/workspace/.beads | head -3')
bd issue update "$I6" --status closed --close-reason "Runtime beads at /home/$TARGET_USER/.openclaw/wo
rkspace/.beads; BEADS_DIR set in .env"
```

## I7: Dotfiles with GNU Stow (optional)

`AskUserQuestion`:

> Git-back your OpenClaw config now? Creates a deploy key, clones your private repo, moves config into
 it, and stows. You'll need the SSH URL ready.
> Options: `yes_setup_dotfiles_now` / `skip_for_now`

If `skip_for_now`:

```bash
bd issue update "$I7" --status closed --close-reason "skipped by user"
```

If `yes_setup_dotfiles_now`, capture `dotfiles_repo_url` into `$INPUTS`, then:

```bash
bd issue update "$I7" --status in_progress

TARGET_USER=$(jq -r '.target_username' "$INPUTS")
REPO_URL=$(jq -r '.dotfiles_repo_url' "$INPUTS")

# 1. Deploy key
sudo -u "$TARGET_USER" -i bash <<EOSU
[ -f ~/.ssh/openclaw_deploy_key ] || ssh-keygen -t ed25519 -f ~/.ssh/openclaw_deploy_key -N "" -C "ope
nclaw-deploy@\$(hostname)"
echo "--- PASTE THE LINE BELOW INTO YOUR REPO'S DEPLOY KEYS SETTINGS (write access) ---"
cat ~/.ssh/openclaw_deploy_key.pub
echo "--- END ---"
EOSU
```

Block via `AskUserQuestion` until the deploy key is registered. Then:

```bash
HOST_ALIAS="github-openclaw"
sudo -u "$TARGET_USER" -i bash <<EOSU
mkdir -p ~/.ssh && chmod 700 ~/.ssh
if ! grep -q "Host $HOST_ALIAS" ~/.ssh/config 2>/dev/null; then
  cat >> ~/.ssh/config <<EOF

Host $HOST_ALIAS
  HostName github.com
  User git
  IdentityFile ~/.ssh/openclaw_deploy_key
  IdentitiesOnly yes
EOF
fi
chmod 600 ~/.ssh/config

ALIAS_URL=\$(echo "$REPO_URL" | sed "s|github.com|$HOST_ALIAS|")
mkdir -p ~/code
[ -d ~/code/dotfiles ] || git clone "\$ALIAS_URL" ~/code/dotfiles

cd ~/code/dotfiles
mkdir -p .openclaw
[ -f ~/.openclaw/openclaw.json ] && [ ! -L ~/.openclaw/openclaw.json ] && mv ~/.openclaw/openclaw.json
 .openclaw/
[ -d ~/.openclaw/cron ] && [ ! -L ~/.openclaw/cron ] && mv ~/.openclaw/cron .openclaw/ 2>/dev/null

[ -f .gitignore ] || cat > .gitignore <<'EOF'
.openclaw/.env
.openclaw/secrets/
.openclaw/workspace/
.openclaw/sessions/
.openclaw/logs/
.openclaw/state/
.openclaw/cache/
.openclaw/memory/[0-9]*.md
EOF

[ -f .stow-local-ignore ] || cat > .stow-local-ignore <<'EOF'
^/\.openclaw/skills($|/)
^/draft-viewer($|/)
EOF

stow --no-folding -t ~ .
ls -la ~/.openclaw/openclaw.json
EOSU

# Skills extraDirs (skill symlinks are rejected by OpenClaw, so point at real path)
sudo -u "$TARGET_USER" -i bash <<'EOSU'
CFG=~/.openclaw/openclaw.json
P="$HOME/code/dotfiles/.openclaw/skills"
TMP=$(mktemp)
jq --arg p "$P" '.skills.load.extraDirs = ((.skills.load.extraDirs // []) + [$p] | unique)' "$CFG" > "
$TMP" && mv "$TMP" "$CFG"
systemctl --user restart openclaw-gateway
EOSU

# Secret scan before commit
sudo -u "$TARGET_USER" -i bash <<'EOSU'
cd ~/code/dotfiles
git add -A
if git diff --cached | grep -E 'sk-ant-|GOCSPX-|ghp_|xoxb-|AKIA|AAG[A-Za-z0-9_-]{20,}'; then
  echo "STOP: secret-shaped string in staged diff. Aborting commit."
  git restore --staged .
  exit 1
fi
git commit -m "OpenClaw initial bootstrap config"
EOSU
```

`AskUserQuestion` on push (do NOT auto-push). Then:

```bash
bd issue update "$I7" --status closed --close-reason "Dotfiles committed; pushed=$(jq -r '.dotfiles_pu
shed // "no"' "$INPUTS")"
```

## I8: SSH tunnel from laptop (informational)

```bash
bd issue update "$I8" --status in_progress
```

This step runs on the user's LAPTOP, not the server. Surface the instructions:

> On your laptop, add to `~/.ssh/config`:
> ```
> Host openclaw
>   HostName <PUBLIC_IP>
>   User <TARGET_USER>
>   LocalForward 18789 localhost:18789
>   LocalForward 6080 localhost:6080
>   ServerAliveInterval 30
> ```
> Then `ssh -fN openclaw` opens both tunnels in the background. Reach the gateway at http://localhost:
18789. Close with `pkill -f "ssh -fN openclaw"`.

Substitute `<PUBLIC_IP>` (from `$INPUTS.public_ip`, captured by `/openclaw-harden`'s preflight, or fet
ch fresh) and `<TARGET_USER>` from `$INPUTS`.

```bash
bd issue update "$I8" --status closed --close-reason "Tunnel instructions surfaced. User configures la
ptop ~/.ssh/config off-box."
```

## I9: First cron job

```bash
bd issue update "$I9" --status in_progress

TARGET_USER=$(jq -r '.target_username' "$INPUTS")
TZ=$(jq -r '.timezone // "Etc/UTC"' "$INPUTS")

sudo -u "$TARGET_USER" -i bash <<EOSU
mkdir -p ~/.openclaw/cron
[ -f ~/.openclaw/cron/jobs.json ] || cat > ~/.openclaw/cron/jobs.json <<EOF
{
  "jobs": [
    {
      "id": "morning-brief",
      "agentId": "main",
      "name": "Morning Brief",
      "enabled": true,
      "schedule": { "kind": "cron", "expr": "0 7 * * *", "tz": "$TZ" },
      "sessionTarget": "isolated",
      "payload": {
        "kind": "agentTurn",
        "message": "Run the morning brief skill.",
        "model": "anthropic/claude-sonnet-4-6",
        "timeoutSeconds": 600
      },
      "delivery": { "mode": "announce", "channel": "last" }
    }
  ]
}
EOF
openclaw cron list
EOSU

bd issue update "$I9" --status closed --close-reason "First cron job 'morning-brief' registered at 0 7
 * * * $TZ"
```

## I10: Final audit

```bash
bd issue update "$I10" --status in_progress

TARGET_USER=$(jq -r '.target_username' "$INPUTS")
FAILS=0

audit_check() {
  local name="$1"; local cmd="$2"
  if eval "$cmd" >/dev/null 2>&1; then
    echo "[PASS] $name"
  else
    echo "[FAIL] $name"
    FAILS=$((FAILS+1))
  fi
}

echo "=== System hardening (re-verify) ==="
audit_check "UFW: 22/tcp LIMIT"               "ufw status verbose | grep -E '22/tcp.*LIMIT'"
audit_check "sshd: PasswordAuthentication no" "grep -RE '^PasswordAuthentication\\s+no' /etc/ssh/"
audit_check "sshd: PermitRootLogin no"        "grep -RE '^PermitRootLogin\\s+no' /etc/ssh/"
audit_check "sshd: AllowUsers $TARGET_USER"   "grep -RE \"^AllowUsers\\s+$TARGET_USER\" /etc/ssh/"
audit_check "sshd -t clean"                   "sshd -t"
audit_check "fail2ban sshd jail"              "fail2ban-client status sshd"
audit_check "unattended-upgrades active"      "systemctl is-active unattended-upgrades"
audit_check "tcp_syncookies = 1"              "[ \"\$(sysctl -n net.ipv4.tcp_syncookies)\" = \"1\" ]"
audit_check "ip_forward = 0"                  "[ \"\$(sysctl -n net.ipv4.ip_forward)\" = \"0\" ]"
audit_check "Linger=yes for $TARGET_USER"     "loginctl show-user $TARGET_USER | grep -q 'Linger=yes'"

echo ""
echo "=== Install state ==="
audit_check "openclaw-gateway active"         "sudo -u $TARGET_USER -i systemctl --user is-active open
claw-gateway | grep -q active"
audit_check "xvfb active"                     "sudo -u $TARGET_USER -i systemctl --user is-active xvfb
 | grep -q active"
audit_check ".env chmod 600"                  "[ \"\$(stat -c %a /home/$TARGET_USER/.openclaw/.env)\" 
= \"600\" ]"
audit_check "gateway bound to 127.0.0.1"      "ss -tlnp 2>/dev/null | grep -E ':18789' | grep -E '127.
0.0.1|::1'"
audit_check "runtime BEADS_DIR set"           "grep -q '^BEADS_DIR=' /home/$TARGET_USER/.openclaw/.env
"

echo ""
echo "=== Blast radius (from inputs) ==="
jq -r '
  "  bot_gmail: " + (.bot_gmail // "NOT_SET"),
  "  bot_social_handles: " + ((.bot_social_handles // "NOT_SET") | tostring),
  "  anthropic_spend_cap_set: " + (.anthropic_spend_cap_set // "unknown"),
  "  never_on_box_confirmed: " + (.never_on_box_confirmed // "unknown")
' "$INPUTS"

echo ""
echo "Failed checks: $FAILS"

if [ "$FAILS" -eq 0 ]; then
  bd issue update "$I10" --status closed --close-reason "All audit checks passed."
else
  bd issue update "$I10" --status blocked --reason "$FAILS audit checks failed"
  echo "Audit found $FAILS failure(s). Fix them, then re-invoke /openclaw-install to re-run."
  exit 1
fi
```

## Close phase and write completion marker

```bash
bd issue update "$I_PHASE" --status closed --close-reason "All I1-I10 sub-issues closed"

EXISTING=$(bd issue list --json 2>/dev/null | jq -r '.[] | select(.title == "install-complete") | .id'
 | head -1)
if [ -z "$EXISTING" ]; then
  IM=$(bd issue create --title "install-complete" --type task --json | jq -r '.id')
  bd issue update "$IM" --status closed --close-reason "OpenClaw installed and configured. Gateway act
ive on loopback, Telegram bot responding, browser stack ready, runtime beads initialized, first cron s
cheduled."
fi

# Optional: archive the bootstrap beads database
bd export --format json > /root/.claude-bootstrap/bootstrap-$(date +%Y%m%d).json 2>/dev/null || true
```

## Done

All three phases complete. Your OpenClaw box is ready.

Expected state:
- `systemctl --user is-active openclaw-gateway` returns `active`
- Gateway bound to `127.0.0.1:18789`
- Telegram bot responding in the target group
- Xvfb active on display `:1`
- Runtime beads initialized at `/home/<user>/.openclaw/workspace/.beads`
- Final audit reports 0 failures
- `install-complete` marker exists and is closed in beads

Reach the gateway from your laptop via `ssh -fN openclaw` (after configuring `~/.ssh/config` per step 
I8).