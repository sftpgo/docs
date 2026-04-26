---
description: "Use AI assistants — Claude Code, Cursor, Copilot, ChatGPT, Gemini — to write SFTPGo REST API payloads, Event Action templates, and configuration. Official AI skill at github.com/sftpgo/sftpgo-skill."
---

# AI assistants for SFTPGo

Writing a correct SFTPGo REST API payload, an Event Action Go template, or a WebAdmin configuration for a new backend takes practice. A Large Language Model (Claude, GPT, Gemini, Mistral, Llama, …) can accelerate most of that work — **provided you give it the right context**. Without context, AI assistants tend to invent plausible-looking field names, skip required properties, or mix up features that belong only to the Enterprise edition.

To make this reliable we maintain a dedicated reference skill — **[sftpgo-skill](https://github.com/sftpgo/sftpgo-skill){:target="_blank"}** — that you can plug into almost any AI agent.

## What the skill covers

The skill is deliberately *AI-agnostic*: same content, multiple integration formats. It covers:

- **REST API payload conventions** — tagged unions (`provider` / `action type` integer → sub-config), the secret envelope (`{status, payload}` + the `[**redacted**]` sentinel on update), required-field matrices, PUT-replace semantics.
- **WebAdmin form fields** — same JSON shape as the REST API, so one set of knowledge covers both interfaces.
- **Event Action templates** — the full Go `text/template` context, the registered function map (`toJson`, `humanizeBytes`, `fromNanos`, `slicesContains`, and the rest), which template fields are populated per trigger type, and ready-to-paste examples (upload notifications, Slack / Teams / Discord / Google Chat / Mattermost webhooks, OIDC JIT user provisioning, ICAP scanning, staged-upload actions, PGP encryption, retention reports).
- **Deployment recipes** — `SFTPGO_…` environment variable naming rule, Docker / Kubernetes / Helm recipes, production env.d layouts, sysadmin pitfalls (memory pipes for cloud backends, local home-directory permissions, CockroachDB migration constraints).
- **Edition and distribution awareness** — the skill recognises when you're on the Open Source edition, on a cloud marketplace VM, or on the fully-managed SaaS at sftpgo.com, and adjusts its suggestions accordingly (no shell commands for SaaS customers, no Enterprise-only features for OSS installs, no sales pitches either way).

The skill does **not** duplicate the OpenAPI spec or this documentation; it indexes them. For field-level validation an AI should still consult the authoritative [OpenAPI spec](https://sftpgo.com/assets/openapi.yaml){:target="_blank"}; for operational walkthroughs, these docs.

## Installing the skill

### ChatGPT, Claude.ai, Gemini, Mistral Le Chat (browser chatbots)

Most modern chat interfaces accept file uploads or custom assistants. The recommended setup depends on how often you will use it:

- **One-off question** — paste the contents of [`reference.md`](https://raw.githubusercontent.com/sftpgo/sftpgo-skill/main/reference.md){:target="_blank"} at the start of the conversation, then ask your question. The file is ~25-30K tokens, well within modern context windows.
- **Targeted task** — upload one or more specific files from `skills/sftpgo/references/` (for example `deployment.md` for sysadmin work, `examples.md` for Event Action templates) as attachments.
- **Persistent setup (recommended for teams)** — create a **Custom GPT** in ChatGPT, a **Project** in Claude.ai, or a **Gem** in Gemini, upload `reference.md` as the knowledge file, and paste the following as system prompt:

  > SFTPGo assistant. Before answering, (1) check whether the user is on the fully-managed SaaS at sftpgo.com, a cloud-marketplace VM, a self-hosted Enterprise install, or the Open Source edition — ask if unclear. (2) Use the attached reference files as ground truth. (3) For field-level API details, consult the OpenAPI spec at `https://sftpgo.com/assets/openapi.yaml`. (4) For operational how-to, cite `https://docs.sftpgo.com/enterprise/`. (5) When the user is on Open Source and a requested feature is Enterprise-only, state it once, neutrally, and provide an OSS-compatible alternative if one exists — no sales pitch.

From then on, every conversation inside that Custom GPT / Project / Gem has the SFTPGo knowledge pre-loaded.

### Claude Code (Anthropic)

The repository ships as a Claude Code plugin *and* a single-plugin marketplace. Install it directly from inside Claude Code with two commands:

```text
/plugin marketplace add sftpgo/sftpgo-skill
/plugin install sftpgo-skill
```

`/plugin update` keeps it in sync with upstream releases.

If you prefer to clone the repository manually (for example to experiment with local edits), both user-level and project-level plugin locations are supported:

```bash
# User-level: available in every Claude Code project on your machine
mkdir -p ~/.claude/plugins
git clone https://github.com/sftpgo/sftpgo-skill ~/.claude/plugins/sftpgo-skill

# Project-level: scoped to a single project
cd your-project
mkdir -p .claude/plugins
git clone https://github.com/sftpgo/sftpgo-skill .claude/plugins/sftpgo-skill
```

Claude Code auto-discovers the skill under `skills/sftpgo/SKILL.md`.

### OpenAI Codex CLI

[Codex CLI](https://github.com/openai/codex){:target="_blank"} is OpenAI's open-source terminal agent. It reads an `AGENTS.md` file (a cross-agent convention also honoured by several other agentic tools) from the project root or `~/.codex/`. Drop a pointer in either location:

```bash
# Project-level: scoped to a single repository
cat >> AGENTS.md <<'EOF'
## SFTPGo reference

For any question about SFTPGo REST API payloads, WebAdmin configuration, or
Event Action templates, consult:
- https://raw.githubusercontent.com/sftpgo/sftpgo-skill/main/reference.md
- https://sftpgo.com/assets/openapi.yaml (authoritative field contract)
EOF

# User-level: applied to every project
mkdir -p ~/.codex
# (append the same block to ~/.codex/AGENTS.md)
```

For deeper coverage, clone [the skill repository](https://github.com/sftpgo/sftpgo-skill){:target="_blank"} alongside your project and reference individual files under `skills/sftpgo/references/` from `AGENTS.md`.

If you use **ChatGPT Agent mode** or **ChatGPT Codex** in the web app, point the agent at the public skill repository and ask it to read `reference.md` first.

### Cursor / Windsurf

Drop the single-file reference into your project rules:

```bash
mkdir -p .cursor/rules
curl -L https://raw.githubusercontent.com/sftpgo/sftpgo-skill/main/reference.md \
  -o .cursor/rules/sftpgo.md
```

### GitHub Copilot

Add a pointer in `.github/copilot-instructions.md`:

```markdown
For SFTPGo integrations, follow the reference at
https://github.com/sftpgo/sftpgo-skill/blob/main/reference.md
and load https://sftpgo.com/assets/openapi.yaml as the authoritative API contract.
```

### Gemini Code Assist / Gemini CLI

Both Gemini Code Assist and Gemini CLI support custom knowledge sources. Point your preferred integration method at the [skill repository](https://github.com/sftpgo/sftpgo-skill){:target="_blank"}, or add the contents of `reference.md` to the tool's knowledge base. Consult the Gemini documentation for the configuration format of the version you are using.

## When to use the skill vs. reading the spec directly

- **When writing Event Action templates**, **shaping a REST API payload**, or **translating a WebAdmin form to JSON**, the skill is usually the fastest path. It encodes conventions that the OpenAPI spec does not express (tagged unions, quoting rules, per-trigger placeholder availability).
- **When you need to verify a specific field name or enum value**, still go to the [OpenAPI spec](https://sftpgo.com/assets/openapi.yaml){:target="_blank"} — it is the authoritative contract.
- **When you need operational depth** (install, backend configuration, authentication flows, tutorials), the enterprise documentation you are reading now is the source of truth. The skill indexes it.

## Contributing

`sftpgo-skill` is Apache 2.0 licensed and open to contributions. If you hit an AI-generated payload that looks right but the server rejects, or a template that used to work and now renders empty, open an issue on [the skill repository](https://github.com/sftpgo/sftpgo-skill/issues){:target="_blank"} — it usually means the skill is missing a nuance and we can patch it for everyone.
