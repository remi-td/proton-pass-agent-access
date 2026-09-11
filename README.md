# Proton Pass Agent Access

A security-first, cross-agent `SKILL.md` for using a named Proton Pass login in an approved workflow without exposing credentials, enumerating a personal vault, or mistaking an email preview for authenticated source material.

## What it covers

- Scoped, time-bounded Proton Pass PAT/agent sessions
- Audited `PROTON_PASS_AGENT_REASON` item reads
- Named login fields: `email`, `username`, `password`, and `url`
- Safe browser-login verification without serializing populated fields
- Complete-member-content verification for research sources
- Incident handling if a credential reaches output, logs, screenshots, chat, or source control

## Install

### Hermes Agent

```bash
hermes skills install https://raw.githubusercontent.com/remi-td/proton-pass-agent-access/main/SKILL.md
```

### skills.sh-compatible agents

```bash
npx skills add remi-td/proton-pass-agent-access
```

### ClawHub / OpenClaw

Publish or install the root `SKILL.md` through the ClawHub CLI when available:

```bash
clawhub skill publish .
```

## Security boundary

This is an instructional skill, not a vault-access tool. It intentionally does not ship a generic credential browser, token, helper, or configuration. The user must approve the specific login and purpose, and the runtime must already have a narrowly scoped Proton Pass session.

## Requirements

- [Proton Pass CLI (`pass-cli`)](https://protonpass.github.io/pass-cli/)
- A user-created, scoped Proton Pass PAT or agent session
- Browser or terminal access appropriate to the approved destination

## Discoverability

Keywords: Proton Pass, pass-cli, agent tokens, PAT, credential safety, least privilege, safe browser automation, subscriber research, secret handling.

## License

[MIT](LICENSE)
