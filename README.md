# openclaw-remote-setup

Three Claude Code skills that bootstrap a fresh Ubuntu 24.04 box into a hardened, production-ready [OpenClaw](https://github.com/openclaw) host. The skills run in sequence and track their own progress in [beads](https://www.npmjs.com/package/@beads/bd) so steps cannot be silently skipped.

## The three skills

Run them in this order. Each skill checks for the prior phase's completion marker in beads and refuses to start if it isn't closed.

| # | Skill | What it does |
|---|---|---|
| 1 | [`/openclaw-prereqs`](skills/openclaw-prereqs/SKILL.md) | Installs Node 24, `jq`, and the `beads` CLI. Initializes the bootstrap beads database at `/root/.claude-bootstrap/.beads` and the shared inputs file at `/root/.openclaw-bootstrap-inputs.json` (mode 600). Writes the `prereqs-complete` marker. |
| 2 | [`/openclaw-harden`](skills/openclaw-harden/SKILL.md) | Hardens the OS: UFW with SSH rate-limit, sshd drop-in (modern Kex/Ciphers/MACs, `AllowUsers`, `MaxAuthTries`), fail2ban local jail, unattended-upgrades with auto-reboot, sysctl kernel tuning, optional TOTP MFA, systemd lingering for the target user. Captures credential blast-radius decisions (dedicated bot Gmail, dedicated social handles, Anthropic spend cap, never-on-box list). Writes the `harden-complete` marker. |
| 3 | [`/openclaw-install`](skills/openclaw-install/SKILL.md) | Installs Claude Code + cc-openclaw skills, OpenClaw gateway as a systemd user service bound to loopback, secrets via `${ENV_VAR}` placeholders, Telegram bot (with the privacy-mode gotcha walkthrough), Xvfb + Chrome headless stack, runtime beads at `~/.openclaw/workspace/.beads`, optional GNU Stow dotfiles, SSH tunnel instructions, first cron job, and a final audit. Writes the `install-complete` marker. |

## Why beads

Long checklists invite agents to skip steps because partially-complete tasks sound plausible. Beads makes skipping structurally impossible: every step is a graph node, blocked by its predecessors, and cannot close without a recorded validation reason. The prereqs skill installs beads before anything else so the later skills inherit this discipline from the first command they run.

## Requirements

- Fresh Ubuntu 24.04 (the skills hard-stop on any other version)
- Root or passwordless sudo
- Network access to the npm registry
- ~10 GB free disk
- Claude Code installed locally, with this repo's `skills/` directory available to it

## Installing the skills

Symlink (or copy) the `skills/` subdirectories into a location Claude Code loads skills from — typically `~/.claude/skills/` for user-scope, or the project-scope `.claude/skills/` if you want them attached to a specific working directory.

```bash
ln -s "$PWD/skills/openclaw-prereqs"  ~/.claude/skills/openclaw-prereqs
ln -s "$PWD/skills/openclaw-harden"   ~/.claude/skills/openclaw-harden
ln -s "$PWD/skills/openclaw-install"  ~/.claude/skills/openclaw-install
```

Then, on the target box, invoke them in order:

```
/openclaw-prereqs
/openclaw-harden
/openclaw-install
```

Each skill has `disable-model-invocation: true` — they only run when invoked explicitly by the user, never automatically by the model.

## Safety rails

The skills include hard stops, not soft suggestions:

- Never reload sshd without a clean `sshd -t`.
- Never `ufw --force enable` unless port 22 is in the ruleset.
- After every sshd change, the user must confirm a second SSH session works in a new terminal before continuing.
- The OpenClaw gateway is bound to `127.0.0.1` only; access is via SSH tunnel.
- Secrets in config files are stored as `${ENV_VAR}` placeholders, never inline; the actual values live in `~/.openclaw/.env` (mode 600).
- Staged git diffs are scanned for secret-shaped strings (`sk-ant-`, `ghp_`, `xoxb-`, `AKIA`, etc.) before any commit.
- Nothing is pushed to a git remote without explicit user confirmation.
- Beads issues only close with validation evidence (actual command output) in the close reason.

## State written to the box

| Path | Purpose |
|---|---|
| `/root/.claude-bootstrap/.beads/` | Bootstrap beads DB tracking the three-skill run |
| `/root/.openclaw-bootstrap-inputs.json` | Persisted parameters (target user, timezone, etc.) — mode 600 |
| `/etc/ssh/sshd_config.d/00-hardening.conf` | sshd hardening drop-in |
| `/etc/fail2ban/jail.local` | fail2ban local jail with sshd enabled |
| `/etc/sysctl.d/99-hardening.conf` | Kernel network hardening |
| `/home/<user>/.openclaw/.env` | Runtime secrets — mode 600 |
| `/home/<user>/.openclaw/workspace/.beads/` | Runtime beads DB for OpenClaw agents |
| `~/.config/systemd/user/openclaw-gateway.service` | Gateway as a user systemd unit |
| `~/.config/systemd/user/xvfb.service` | Headless X server on `:1` |

## License

[MIT](LICENSE)
