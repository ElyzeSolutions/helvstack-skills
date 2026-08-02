# Helvstack Agent Skills

Public, portable [Agent Skills](https://agentskills.io/) for deploying applications on [Helvstack](https://helvstack.com/).

## Install

Install the deployment skill for every supported coding agent detected on your machine:

```bash
npx skills add ElyzeSolutions/helvstack-skills --skill helvstack-agent-deploy -g -y
```

Target one harness explicitly when needed:

```bash
npx skills add ElyzeSolutions/helvstack-skills --skill helvstack-agent-deploy -g -a codex -y
npx skills add ElyzeSolutions/helvstack-skills --skill helvstack-agent-deploy -g -a claude-code -y
```

Inspect the available skills without installing:

```bash
npx skills add ElyzeSolutions/helvstack-skills --list
```

## Skills

- `helvstack-agent-deploy`: inspect a repository, obtain least-privilege Helvstack access, plan changes, apply them idempotently, wait for terminal states, and return deployment evidence.

The canonical human and machine onboarding guide is at [helvstack.com/agent](https://helvstack.com/agent).

## Source of truth

This repository publishes the public skill surface only. Helvstack maintains the canonical skill alongside its product contracts and verifies that its repository and hosted copies remain byte-for-byte identical before release.
