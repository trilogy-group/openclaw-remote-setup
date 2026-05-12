---
name: openclaw-harden
description: Harden a fresh Ubuntu 24.04 box for production OpenClaw use. UFW rate-limit, sshd hardening (modern KexAlgorithms/Ciphers/MACs, AllowUsers, MaxAuthTries), fail2ban with a local jail, unattended-upgrades with auto-reboot, sysctl kernel tuning, optional TOTP MFA on SSH, systemd lingering for the target user, plus a structured credential blast-radius conversation (dedicated bot Gmail, dedicated social handles, spend caps, never-on-this-box list). Tracked in beads. Skill 2 of 3. Requires /openclaw-prereqs to have run first.
disable-model-invocation: true
---

# OpenClaw bootstrap, phase 2: hardening

This is skill 2 of 3. It hardens the OS and walks the user through the credential blast-radius decisio
ns before any application code touches the box.

Order:
1. `/openclaw-prereqs` (prerequisite)
2. `/openclaw-harden` (this skill)
3. `/openclaw-install`

## Preflight: confirm phase 1 ran

```bash
INPUTS=/root/.openclaw-bootstrap-inputs.json
BEADS_HOME=/root/.claude-bootstrap

[ -d "$BEADS_HOME/.beads" ] || { echo "STOP: bootstrap beads not initialized. Run /openclaw-prereqs fi
rst."; exit 1; }
[ -f "$INPUTS" ] || { echo "STOP: inputs file missing. Run /openclaw-prereqs first."; exit 1; }
command -v bd >/dev/null 2>&1 || { echo "STOP: beads CLI missing. Run /openclaw-prereqs first."; exit 
1; }
command -v jq >/dev/null 2>&1 || { echo "STOP: jq missing. Run /openclaw-prereqs first."; exit 1; }

cd "$BEADS_HOME"
MARKER=$(bd issue list --status closed --json 2>/dev/null | jq -r '.[] | select(.title == "prereqs-com
plete") | .id' | head -1)
[ -n "$MARKER" ] || { echo "STOP: prereqs-complete marker not found/closed. Run /openclaw-prereqs firs
t."; exit 1; }
echo "Preflight OK. Prereqs marker: $MARKER"
```

## Critical safety rules

1. **Never reload sshd without first running `sshd -t`.** If `sshd -t` exits non-zero, do not proceed.
2. **Never run `ufw --force enable` without first confirming `ufw status` lists port 22.** If 22 is mi
ssing, the user will be locked out.
3. **After every change to `/etc/ssh/sshd_config`, ask the user to open a SECOND SSH session in a new 
terminal and confirm it works** before continuing.
4. **If any step's validation fails, stop.** Set the beads issue to `blocked` with the error and do no
t silently continue.
5. **Do not close a beads issue without recording validation evidence in the close reason.** Paste the
 actual command output.

## Inputs to collect

| Input | When | Default | Persist as |
|---|---|---|---|
| target_username | Step H1 | `openclaw` | `.target_username` |
| timezone | Step H2 | `Etc/UTC` | `.timezone` |
| auto_reboot_time | Step H2 | `03:30` | `.auto_reboot_time` |
| enable_totp_mfa | Step H8 | `yes` | `.enable_totp_mfa` |
| anthropic_spend_cap_set | Step H10 | `yes` | `.anthropic_spend_cap_set` |
| bot_gmail | Step H10 | none | `.bot_gmail` |
| bot_social_handles | Step H10 | none | `.bot_social_handles` |
| never_on_box_confirmed | Step H10 | `yes` | `.never_on_box_confirmed` |

Use `AskUserQuestion`. Never echo secrets back; show `prefix...suffix` only.

## Step graph: create or reuse

Idempotent helper. Adapt flag names to whatever `bd issue create --help` shows.

```bash
cd /root/.claude-bootstrap
MAP=/root/.claude-bootstrap/harden-ids.env

ensure_issue() {
  # $1: var name, $2: title, $3..: blocked-by var names from this scope
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

PREREQ_MARKER=$(bd issue list --status closed --json | jq -r '.[] | select(.title == "prereqs-complete
") | .id' | head -1)

ensure_issue H_PHASE "harden-phase" 
ensure_issue H1  "Harden 1: system updates and core packages" PREREQ_MARKER
ensure_issue H2  "Harden 2: unattended-upgrades with auto-reboot" H1
ensure_issue H3  "Harden 3: UFW with SSH rate limiting" H1
ensure_issue H4  "Harden 4: create non-root user" H1
ensure_issue H5  "Harden 5: harden sshd (drop-in, validate, reload)" H4
ensure_issue H6  "Harden 6: fail2ban with local jail" H1
ensure_issue H7  "Harden 7: sysctl kernel hardening" H1
ensure_issue H8  "Harden 8: TOTP MFA on SSH (optional)" H5
ensure_issue H9  "Harden 9: systemd lingering for target user" H4
ensure_issue H10 "Harden 10: credential blast-radius decisions" H5
# Phase issue blocked by all sub-steps
bd issue update "$H_PHASE" --blocked-by "$H1" --blocked-by "$H2" --blocked-by "$H3" --blocked-by "$H4"
 \
    --blocked-by "$H5" --blocked-by "$H6" --blocked-by "$H7" --blocked-by "$H8" --blocked-by "$H9" --b
locked-by "$H10"

bd ready
```

After this, `bd ready` should show H1 as the next runnable.

## Tracking pattern

For each step H1-H10:
1. `bd ready` confirms it is next.
2. `bd issue update <ID> --status in_progress`
3. Run the commands.
4. Run the validation.
5. On success: `bd issue update <ID> --status closed --close-reason "<paste output>"`.
6. On failure: `bd issue update <ID> --status blocked --reason "<error>"` and STOP.

---

## H1: System updates and core packages

```bash
bd issue update "$H1" --status in_progress

apt update && apt -y upgrade
apt install -y unattended-upgrades ufw fail2ban update-notifier-common libpam-google-authenticator

EV=$(dpkg -l unattended-upgrades ufw fail2ban update-notifier-common libpam-google-authenticator | gre
p -c '^ii')
[ "$EV" -ge 5 ] || { bd issue update "$H1" --status blocked --reason "only $EV/5 packages installed"; 
exit 1; }

bd issue update "$H1" --status closed --close-reason "5/5 packages installed: unattended-upgrades, ufw
, fail2ban, update-notifier-common, libpam-google-authenticator"
```

## H2: Unattended-upgrades with auto-reboot

`AskUserQuestion` for `auto_reboot_time` and `timezone` if not in `$INPUTS`. Persist before proceeding
.

```bash
bd issue update "$H2" --status in_progress

REBOOT_TIME=$(jq -r '.auto_reboot_time // "03:30"' "$INPUTS")
CONF=/etc/apt/apt.conf.d/50unattended-upgrades

set_key() {
  local key="$1"; local val="$2"
  if grep -qE "^//*${key}" "$CONF"; then
    sed -i "s|^//*${key}.*|${key} ${val};|" "$CONF"
  else
    echo "${key} ${val};" >> "$CONF"
  fi
}
set_key 'Unattended-Upgrade::Automatic-Reboot' '"true"'
set_key 'Unattended-Upgrade::Automatic-Reboot-Time' "\"$REBOOT_TIME\""
set_key 'Unattended-Upgrade::Automatic-Reboot-WithUsers' '"false"'
set_key 'Unattended-Upgrade::Remove-Unused-Kernel-Packages' '"true"'
set_key 'Unattended-Upgrade::Remove-Unused-Dependencies' '"true"'

dpkg-reconfigure --priority=low unattended-upgrades
systemctl enable --now unattended-upgrades.service

OUT=$(grep -E '^Unattended-Upgrade::Automatic-Reboot ' "$CONF")
[ -n "$OUT" ] || { bd issue update "$H2" --status blocked --reason "Automatic-Reboot directive missing
"; exit 1; }
bd issue update "$H2" --status closed --close-reason "$OUT; service=$(systemctl is-active unattended-u
pgrades)"
```

## H3: UFW with SSH rate limiting

```bash
bd issue update "$H3" --status in_progress

ufw default deny incoming
ufw default allow outgoing
ufw limit 22/tcp

PRE=$(ufw status verbose | grep -E '22/tcp.*LIMIT' || true)
[ -n "$PRE" ] || { bd issue update "$H3" --status blocked --reason "22/tcp LIMIT rule missing pre-enab
le"; exit 1; }

ufw --force enable
OUT=$(ufw status verbose | head -20)
bd issue update "$H3" --status closed --close-reason "$OUT"
```

## H4: Create non-root user

`AskUserQuestion` for `target_username` (default `openclaw`). Persist.

```bash
bd issue update "$H4" --status in_progress

TARGET_USER=$(jq -r '.target_username // "openclaw"' "$INPUTS")
if ! id "$TARGET_USER" >/dev/null 2>&1; then
  adduser --disabled-password --gecos "" "$TARGET_USER"
  usermod -aG sudo "$TARGET_USER"
  if [ -d /root/.ssh ]; then
    rsync -a /root/.ssh "/home/$TARGET_USER/"
    chown -R "$TARGET_USER:$TARGET_USER" "/home/$TARGET_USER/.ssh"
  fi
  PW=$(openssl rand -base64 32)
  echo "${TARGET_USER}:${PW}" | chpasswd
fi
OUT=$(id "$TARGET_USER")
bd issue update "$H4" --status closed --close-reason "$OUT"
```

## H5: Harden sshd (drop-in config, validate, reload, confirm second session)

```bash
bd issue update "$H5" --status in_progress

TARGET_USER=$(jq -r '.target_username // "openclaw"' "$INPUTS")
cp /etc/ssh/sshd_config "/etc/ssh/sshd_config.bak.$(date +%s)"

cat > /etc/ssh/sshd_config.d/00-hardening.conf <<EOF
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
PermitEmptyPasswords no
MaxAuthTries 3
LoginGraceTime 20
AllowUsers $TARGET_USER

KexAlgorithms sntrup761x25519-sha512@openssh.com,curve25519-sha256,curve25519-sha256@libssh.org,diffie
-hellman-group16-sha512,diffie-hellman-group18-sha512
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

X11Forwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
EOF

if ! sshd -t 2>&1; then
  bd issue update "$H5" --status blocked --reason "sshd -t failed"
  exit 1
fi
systemctl reload ssh
```

Then BLOCK on user confirmation via `AskUserQuestion`:

> Open a NEW terminal and run: `ssh <TARGET_USER>@<PUBLIC_IP>`. Confirm the second session connects be
fore we continue.
> Options: `confirmed_second_session_works` / `it_fails_help`

If `it_fails_help`, do not close the issue. Help diagnose without losing the first session. Common fix
es:
- Trying to log in as root (must use `<TARGET_USER>`)
- Key not copied to new user's `~/.ssh/authorized_keys`

On confirmation:

```bash
bd issue update "$H5" --status closed --close-reason "sshd -t clean, second session confirmed, AllowUs
ers=$TARGET_USER"
```

## H6: fail2ban with local jail

```bash
bd issue update "$H6" --status in_progress

cat > /etc/fail2ban/jail.local <<'EOF'
[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 5
backend  = systemd

[sshd]
enabled = true
EOF

systemctl enable --now fail2ban
sleep 2
OUT=$(fail2ban-client status sshd 2>&1) || { bd issue update "$H6" --status blocked --reason "$OUT"; e
xit 1; }
bd issue update "$H6" --status closed --close-reason "$OUT"
```

## H7: sysctl kernel hardening

```bash
bd issue update "$H7" --status in_progress

cat > /etc/sysctl.d/99-hardening.conf <<'EOF'
net.ipv4.ip_forward = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
kernel.yama.ptrace_scope = 2
kernel.kptr_restrict = 2
EOF

sysctl -p /etc/sysctl.d/99-hardening.conf
OUT=$(sysctl net.ipv4.tcp_syncookies net.ipv4.ip_forward net.ipv4.conf.all.rp_filter)
[ "$(sysctl -n net.ipv4.tcp_syncookies)" = "1" ] || { bd issue update "$H7" --status blocked --reason 
"tcp_syncookies not 1"; exit 1; }
bd issue update "$H7" --status closed --close-reason "$OUT"
```

## H8: TOTP MFA on SSH (optional)

`AskUserQuestion`:
> Enable TOTP two-factor auth on SSH? Recommended for a long-lived public box.
> Options: `enable_totp` / `skip_for_now`

If `skip_for_now`:

```bash
bd issue update "$H8" --status closed --close-reason "skipped by user"
```

If `enable_totp`:

The `google-authenticator` command is interactive and cannot be piped. Surface it to the user:

```bash
TARGET_USER=$(jq -r '.target_username // "openclaw"' "$INPUTS")
echo "Run this NOW as $TARGET_USER in a separate terminal. Save the backup codes to a password manager
 (NOT this server):"
echo "  sudo -u $TARGET_USER -i google-authenticator"
echo "Answer y to all prompts. Scan the QR into your authenticator app."
```

Wait for `AskUserQuestion` confirmation that they ran it and saved the backup codes. Then:

```bash
bd issue update "$H8" --status in_progress

if ! grep -q 'pam_google_authenticator.so' /etc/pam.d/sshd; then
  sed -i '1i auth required pam_google_authenticator.so' /etc/pam.d/sshd
fi

cat >> /etc/ssh/sshd_config.d/00-hardening.conf <<'EOF'

KbdInteractiveAuthentication yes
AuthenticationMethods publickey,keyboard-interactive
EOF

sshd -t || { bd issue update "$H8" --status blocked --reason "sshd -t failed after MFA config"; exit 1
; }
systemctl reload ssh
```

Block via `AskUserQuestion` on a third SSH session confirming both factors prompt. If yes:

```bash
bd issue update "$H8" --status closed --close-reason "TOTP enabled, third session confirmed both facto
rs prompt"
```

## H9: systemd lingering for target user

```bash
bd issue update "$H9" --status in_progress

TARGET_USER=$(jq -r '.target_username // "openclaw"' "$INPUTS")
loginctl enable-linger "$TARGET_USER"
OUT=$(loginctl show-user "$TARGET_USER" | grep Linger)
[ "$OUT" = "Linger=yes" ] || { bd issue update "$H9" --status blocked --reason "Linger not yes: $OUT";
 exit 1; }
bd issue update "$H9" --status closed --close-reason "$OUT"
```

## H10: Credential blast-radius decisions

No shell. Structured conversation. Ask via `AskUserQuestion`, persist each answer to `$INPUTS`.

```bash
bd issue update "$H10" --status in_progress
```

1. **Anthropic spend cap.**
   > Have you set a monthly spend cap on your Anthropic API key at console.anthropic.com?
   > Options: `yes_set` / `not_yet_will_do_now` / `decline`
   If `not_yet_will_do_now`, pause until they confirm. If `decline`, log a strong warning.

2. **Dedicated bot Gmail.**
   > Will this box use a dedicated bot Google account (recommended) or your personal Gmail?
   > Options: `dedicated_bot_account` / `personal_account_understood_risk` / `no_google_account`
   Capture email into `inputs.bot_gmail`. If personal, log a strong warning that an agent bug or promp
t injection can move/share/delete personal email or Drive content.

3. **Dedicated social handles.**
   > For X, LinkedIn, Reddit, Instagram, Bluesky: dedicated bot handles?
   > Options: `all_dedicated` / `some_personal_some_bot` / `all_personal_understood_risk` / `no_social
_platforms`
   Capture into `inputs.bot_social_handles`.

4. **Never-on-this-box.**
   > Confirm none of these will be placed on this box: production SSH keys, banking creds, payment car
ds, wallet seeds, password manager exports, AWS root keys, GCP owner creds, personal MFA backup codes,
 production DB passwords.
   > Options: `confirmed_will_not_place_any` / `need_to_audit_first`

Then close:

```bash
bd issue update "$H10" --status closed --close-reason "$(jq -c '{spend_cap: .anthropic_spend_cap_set, 
bot_gmail: .bot_gmail, social: .bot_social_handles, never_on_box: .never_on_box_confirmed}' "$INPUTS")
"
```

## Close the phase and write the completion marker

```bash
cd /root/.claude-bootstrap

# Phase auto-becomes ready once all sub-issues close. Confirm and close it.
bd issue update "$H_PHASE" --status closed --close-reason "All H1-H10 sub-issues closed"

# Completion marker for /openclaw-install to check
EXISTING=$(bd issue list --json 2>/dev/null | jq -r '.[] | select(.title == "harden-complete") | .id' 
| head -1)
if [ -z "$EXISTING" ]; then
  HM=$(bd issue create --title "harden-complete" --type task --json | jq -r '.id')
  bd issue update "$HM" --status closed --close-reason "Hardening phase done. UFW limit on 22, sshd ha
rdened, fail2ban active, sysctl tuned, lingering on for $(jq -r .target_username "$INPUTS"). Blast-rad
ius decisions captured."
fi

bd issue list --status closed --json | jq -r '.[] | select(.title == "harden-complete") | "Marker: \(.
id) - \(.title)"'
```

## Done

Phase 2 complete. Run `/openclaw-install` next.

Expected state:
- UFW shows `22/tcp LIMIT`
- `sshd -t` exits clean
- `fail2ban-client status sshd` returns OK
- `unattended-upgrades` service is active
- `sysctl net.ipv4.tcp_syncookies` returns 1
- `loginctl show-user <user>` returns `Linger=yes`
- `harden-complete` issue exists and is closed in beads