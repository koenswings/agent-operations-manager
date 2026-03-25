> **Question:** Regarding openclaw itself. As this is the main tool to manage the idea organisation and as the idea repo represents the digital footprint of the idea organisation, shouldn't we merge the openclaw folder into the idea repo? What do you think about this?

# Should openclaw be merged into the idea repo?

Good question, and worth thinking through carefully.

## The intuition is right, but a full merge has a trap

`openclaw.json` contains bearer tokens and channel credentials (WhatsApp pairing,
`LOCAL_AUTH_TOKEN` for Mission Control). Those must never enter the `idea` git history
— not even once, because git history is forever. A full merge creates ongoing risk of
accidentally committing secrets alongside org content.

## The more useful distinction is: platform vs. configuration

OpenClaw is a dependency — like Docker or Node. The `openclaw` repo holds the platform:
its binary, compose setup, volumes, upgrade history. That doesn't belong in `idea`.

But *how IDEA configures OpenClaw* — the agent roster, workspace paths, heartbeat
schedules, channel policies — that IS org content. It describes the virtual company's
structure just as much as `ROLES.md` does.

## What I'd recommend instead

Keep `openclaw` as a separate repo. But add an `openclaw/` folder to the `idea` repo
that holds the IDEA-specific configuration as committed, secrets-free reference files:

```
idea/
  openclaw/
    openclaw.json          ← agent roster, workspaces, heartbeats (no tokens)
    compose-additions.yaml ← Mission Control service block to add to compose.yaml
    README.md              ← how to apply these to the live openclaw instance
```

Tokens and credentials stay out of git entirely — set once in the live `openclaw.json`
on disk and never committed.

This gives you what the merge was trying to achieve: the `idea` repo is the single source
of truth for how the organisation is configured. But it avoids the secrets risk and keeps
platform upgrades out of the org history.

## The practical trigger

This is most relevant when we deploy the full agent roster — the `openclaw.json` that
defines all six agents, their workspaces, heartbeat schedules, and channel policies should
live in `idea/openclaw/` from day one, minus credentials. Anyone looking at the `idea` repo
can see exactly how OpenClaw is configured to run the virtual company, without any secrets
being exposed.

## Summary

| Approach | Pros | Cons |
|----------|------|------|
| Full merge | One repo for everything | Secrets risk; platform noise in org history |
| Keep separate | Clean separation | Config lives outside org repo |
| **Recommended: `idea/openclaw/` subfolder** | **Org config in idea repo, no secrets, platform stays separate** | **Slight duplication between reference and live file** |
