# rakazo-self-host

An Agent Skill (`SKILL.md`) for installing, configuring, verifying, maintaining and troubleshooting a **self-hosted Rakazo server** on macOS (Docker published images + Tailscale HTTPS).

It covers preflight checks, `.env` URLs, OpenRouter / Composio keys (use the Composio **Platform** `ak_` project key, not the `ck_` Connect key), first-account creation (email first, or `bootstrap-owner`), Gmail SMTP / Resend email, upgrades, backups and common pitfalls.

Companion macOS app: [rakazo-server-app](https://github.com/agent-dailywork/rakazo-server-app)

## Install

- **Rakazo**: create a skill and paste `SKILL.md`, or ask your bot to import it.
- **Claude Code / other Agent Skills runtimes**: copy this folder into your skills directory, e.g.

```sh
git clone https://github.com/agent-dailywork/rakazo-self-host ~/.claude/skills/rakazo-self-host
```

The skill contains no accounts, hostnames or keys. Never paste secrets into chat; the skill checks keys by prefix / HTTP status only.
