# powerbi-app-links

A Claude Code skill for fetching Power BI App metadata (Apps, Reports,
Dashboards, and direct links into them) via the official Power BI REST API,
plus a clearly-flagged best-effort path for Audience data via an
undocumented endpoint.

## Install

Copy `SKILL.md` into your skills directory (e.g. `~/.claude/skills/powerbi-app-links/SKILL.md`
for Claude Code, or the equivalent skills folder for whichever Claude
surface you're using).

## Setup

You'll need an Entra ID app registration configured as a Power BI service
principal (see `SKILL.md` for the exact tenant settings and permissions
required), then set:

```bash
export POWERBI_TENANT_ID=...
export POWERBI_CLIENT_ID=...
export POWERBI_CLIENT_SECRET=...
```

See `.env.example` for a template.

## What it does and doesn't cover

- ✅ Apps, Reports, Dashboards, and their direct `webUrl` links — via the
  official, documented REST API.
- ⚠️ Audiences — no documented API exists; this skill includes an optional,
  explicitly opt-in, best-effort path using an undocumented internal
  endpoint that requires a manually-obtained browser session token. It is
  unsupported and can break at any time.
- ❌ Sections (App navigation groupings) — no API, documented or
  undocumented, currently exposes this at all. Not attempted.

Full details, including auth flow and endpoint specifics, are in `SKILL.md`.

## License / disclaimer

This is not affiliated with or endorsed by Microsoft. The undocumented
endpoint referenced here is unsupported, may violate Microsoft's terms of
service depending on your usage, and may stop working without notice. Use
at your own risk, particularly in production or client-facing automation.
