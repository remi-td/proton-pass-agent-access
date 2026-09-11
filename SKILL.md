---
name: proton-pass-agent-access
description: Use when agents access Proton Pass logins safely.
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [proton-pass, pass-cli, credentials, secrets, agents, security]
    related_skills: [local-cli-bootstrap, web-provider-integration]
  openclaw:
    requires:
      bins: [pass-cli]
---

# Proton Pass Agent Access

## Overview

Use Proton Pass CLI (`pass-cli`) to let an agent perform a specific, user-approved action with a named credential, without turning the entire vault into a model-accessible data source.

This skill is for **using** an existing login, not for general vault administration. It treats a Proton Pass personal access token (PAT) or agent token as a least-privilege capability: it should be limited to the one vault or item required, have Viewer access unless writing is explicitly needed, expire automatically, and leave an auditable reason for each item read.

## When to Use

Use when an agent must:

- sign in to a user-approved website using a named Proton Pass login;
- retrieve one approved secret for a defined integration;
- diagnose why a scoped Proton Pass agent session cannot read an expected login;
- prepare a safe, repeatable credential-access workflow.

Do **not** use when:

- the user has not approved the exact service/login and purpose;
- broad vault exploration is the proposed way to discover credentials;
- the task requires an SSH private key, recovery phrase, account password, or unrestricted personal vault access;
- a credential is already known or has been exposed in logs, chat, source control, or diagnostic output. Handle that as an incident first.

## Security Model

### Separate identities and scopes

Use one of these models:

1. **Interactive human session:** the user authenticates in their own browser. Suitable for setup and vault administration, not unattended automation.
2. **Scoped agent/PAT session:** the user creates a time-bounded token with access to one designated vault or item. Suitable for an agent performing the approved integration.
3. **Purpose-specific wrapper:** for recurring work, a reviewed wrapper retrieves only the named fields and performs only the intended action. Do not provide a generic vault lookup tool to the model.

A Viewer grant can still read every secret in its scope. Scope the token to the smallest practical vault or item; do not confuse Viewer with field-level secrecy.

### Required audit reason

Proton Pass agent sessions require `PROTON_PASS_AGENT_REASON` for audited item reads. Set a concise, accurate reason for every `item view`; it becomes part of the encrypted audit record visible to the vault owner.

Example reason:

```bash
export PROTON_PASS_AGENT_REASON='Sign in to the approved research portal and read the requested subscriber article.'
```

Never put a secret, email address, username, or tracking URL in that reason.

## Setup: Human-Performed Once

The user should create an agent/PAT from an interactive Proton Pass session, set an expiration, and grant only the intended vault or item at Viewer role. Proton Pass documentation supports vault-level or item-level grants; item-level is preferred where it does not make the workflow brittle.

Store the token in a `0600` file owned by the dedicated service account that consumes it, never in shell startup files, source code, cron prompts, or a general `.env` file. Use a dedicated `PROTON_PASS_SESSION_DIR` with `0700` permissions.

The agent session is short-lived. Before a task, test it with `pass-cli info`; if expired, refresh it from the separately protected token file without printing either token or account metadata.

## Named Login Retrieval Workflow

### 1. Confirm scope before reading

Obtain from the user or task context:

- the named login entry;
- the destination website/domain;
- the concrete purpose (for example, read one subscriber report);
- whether a full member page—not merely a forwarded email or preview—is required.

Do not use an unbounded `vault list`, `item list`, or bulk export as a discovery mechanism. A user may explicitly authorise a narrow title lookup during one setup/repair task, but keep titles and all resulting metadata out of model-visible output.

### 2. Read only required standard fields

A Proton Pass **login** item can contain `email`, `username`, `password`, and `url`. Retrieve only fields needed by the site; use `email` when the site asks for an email, not `username` by assumption.

```bash
PROTON_PASS_AGENT_REASON='Sign in to the approved research portal and read the requested subscriber article.' \
pass-cli item view \
  --vault-name 'Approved Agent Vault' \
  --item-title 'Approved Research Portal' \
  --field email
```

Use the same named item and a separate field read for `password` and, if needed, `url`. Prefer exact `--vault-name` + `--item-title` or a pre-approved `pass://` reference. Do not print command output, capture it in a transcript, or pass it as a command-line argument to another process.

A safe integration keeps each retrieved value in process memory only, fills it directly into the intended browser form, and discards it after login. It must not write credentials to temporary files, environment dumps, shell history, logs, chat, code, or persistent configuration.

### 3. Verify authentication without inspecting secrets

After submitting the login form, verify one or more non-secret signals:

- the authenticated page or account-only navigation is present;
- the login form is absent;
- the expected member article/transcript is accessible;
- a signed-in HTTP response or page title confirms access.

Never dump `document.body.innerText` before checking whether it contains user/account data. Never serialize an input element, inspect `.value`, use `innerText || value`, capture a screenshot with populated credentials, or log browser network requests containing login data.

### 4. Verify source completeness

For research subscriptions, an announcement, email preview, episode page, or search snippet is not enough. Confirm that the full member article/PDF/transcript is loaded, then read the complete source before treating its detailed claims as digest material. Record the access basis in the source note: authenticated full content, user-provided export, or email-only preview.

## Browser and Credential Handling Rules

- Navigate only to the credential item's approved URL or the user-approved destination domain. Treat redirects to a different registrable domain as a stop-and-review condition.
- Fill fields using selectors; inspect only selectors, labels, non-secret attributes, and login-state booleans.
- Do not enable “remember me” unless the user explicitly asks. Clear populated fields if login cannot proceed.
- Do not retrieve or submit TOTP/recovery data without a separate explicit approval. Prefer the site’s existing trusted session or user-operated MFA.
- Do not reuse a retrieved password against another domain or account.
- Do not weaken browser security, bypass paywalls, or automate account recovery.

## Troubleshooting

### The item appears unavailable

Check these in order without printing item contents:

1. `pass-cli info` confirms a live session.
2. The agent/PAT has not expired; sessions themselves are short-lived.
3. `PROTON_PASS_AGENT_REASON` is set for item reads.
4. The lookup specifies both the intended vault/share and the named item, or uses its approved secret reference.
5. The token grant covers the intended vault/item at Viewer role or higher.
6. The field requested matches the login field actually required by the website: `email`, `username`, `password`, or `url`.

Do not conclude that an entry is password-only merely because an unaudited or incorrectly scoped field read failed.

### Login fails despite a successful field read

Stop after one normal attempt. Check the credential URL/domain, the field labels, and whether the account requires user-operated MFA. Do not repeat attempts rapidly, invoke password-reset flows, or keep the credentials in the browser while investigating.

### Full content remains inaccessible

Label the source as email-only or preview-only. Do not infer missing tables, rankings, transcript passages, or trade expressions. Ask the user for a member export or for approval to resolve the access issue.

## Incident Response

If a credential appears in terminal output, browser diagnostics, screenshots, chat, source control, or logs:

1. Stop using it and clear any browser fields.
2. Tell the user what type of credential was exposed without repeating its value.
3. Rotate/revoke it at the affected website and update the Proton Pass login only through a user-approved path.
4. Remove persisted copies where possible and verify the replacement is scoped correctly.
5. Record the failure mode as a non-secret lesson in the relevant skill or integration runbook.

## Common Pitfalls

1. **Treating one accessible vault share as one accessible item.** A vault-level Viewer grant can contain multiple approved items; inspect the token grant and use a named lookup.
2. **Omitting `PROTON_PASS_AGENT_REASON`.** Item reads can fail or behave differently; set it before testing field access.
3. **Assuming `username` when the login has `email`.** Read the site label and request the matching standard field.
4. **Inspecting a filled browser form.** Diagnostics that include input values can disclose a password. Inspect non-secret state only.
5. **Calling an email announcement “full-source research.”** Verify the complete authenticated item actually loaded before using member-only details.

## Verification Checklist

- [ ] The user approved the named service/login and the purpose.
- [ ] The scoped session is live and uses a dedicated session directory.
- [ ] Each `item view` has a concise, non-secret audit reason.
- [ ] Only required named fields were read; no bulk vault/item data was exposed.
- [ ] The browser login was verified with non-secret signals.
- [ ] The full member content, not a preview, was confirmed before research use.
- [ ] No secret was printed, persisted, or included in diagnostics.
- [ ] A failure is honestly labelled as unavailable, preview-only, or email-only.
