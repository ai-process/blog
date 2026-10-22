---
layout: post
title: "LiteLLM 1.107.0 shipped a credential stealer: what to check and what to rotate"
date: 2026-10-21 19:08:00 +0000
reading_time: 5
tags:
  - security
  - supply-chain
  - litellm
description: "A malicious litellm 1.107.0 was published to PyPI and Docker registries on 2026-10-21. A practical checklist: how to tell if you are affected, how to clean up, and which secrets to rotate."
---

On 2026-10-21, `litellm==1.107.0` was published to PyPI with a credential stealer inside every one
of its compiled wheels. Container images tagged `v1.107.0` were affected too. If you run LiteLLM anywhere, this is a checklist.

**Updated 2026-10-22** with BerriAI's [incident report](https://docs.litellm.ai/blog/security-incident-oct-21-2026)
(its 20:30 UTC update): exact times, how the release got out, more files to look for and more secrets to
rotate. The investigation is still open, so check the report for later updates before acting.

---

## What happened

Times are UTC, 2026-10-21, from BerriAI's incident report and
[StepSecurity](https://www.stepsecurity.io/blog/litellm-1-107-0-compromised-pypi-backdoored-wheels-credential-stealer):

| Time | Event |
|---|---|
| 08:56–10:46 | Payload commits pushed to a lookalike repository, `Berri-AI/litellm` (the real one is `BerriAI/litellm`) |
| 10:08 | Branch protection blocks a change to the private release workflow; a privileged developer credential overrides it |
| 10:46 | The modified workflow builds from the lookalike repository |
| 11:11 | A second developer account approves the production release |
| 11:39 | Seven `litellm==1.107.0` wheels uploaded to PyPI |
| 11:46 | Infected images published; `latest`, `main-latest` and `main-stable` point to them |
| 11:53 | A third developer account deletes the workflow run records |
| 12:06 | Compromise reported in BerriAI/litellm issue #47826 |
| by 13:23 | `1.107.0` removed from PyPI |
| 13:51 | Affected GHCR tags no longer available |
| 14:20 | All affected Docker Hub tags removed or pointed back to clean images |

### How it got out

According to BerriAI, the attacker held GitHub CLI credentials of three developer accounts: one
changed and ran the release workflow, one approved the production release, and one deleted the
run records. A token of a GitHub organization owner was used to turn off branch protection. The
official `v1.107.0` source tag does not contain the malicious files; the artifacts were built from
the lookalike repository. How the attacker got the tokens in the first place is not yet confirmed.

### What is affected

* **PyPI:** `litellm==1.107.0`, all seven wheels (macOS, Linux glibc and musl, Windows). The sdist
  has a tampered build config but no payload.
* **Docker:** the `v1.107.0` tag of `ghcr.io/berriai/litellm`, `litellm-database`,
  `litellm-non_root` and `docker.io/litellm/litellm`. The floating tags `latest`, `main-latest` and
  `main-stable` pointed to the infected image from 11:46 to 14:20 UTC, about two and a half hours.
  The Docker Hub index digest is
  `sha256:1bbcb1443bc962e45f5aad53fe3d2c30600e0379715fb6c664cdd3ac126c339e`.
* **Reported clean:** BerriAI says every earlier version is safe; StepSecurity also checked
  `1.107.0rc1`, `1.107.0rc2` and the `1.103.x`–`1.106.x` releases.

### What the malware does

It installs three files into `site-packages`: `cook.pth`, `cook.py` and `sitecustomize.py`.
`cook.pth` imports `cook`, so the stealer runs on **every Python interpreter start** in that
environment. Your code does not have to import `litellm`.

It collects browser data, SSH and GPG keys, cloud and CI credentials, `.env` files, the environment
variables of readable processes and tokens of AI coding tools. If `DATABASE_URL` is set, it also
dumps the LiteLLM proxy database, which holds upstream provider API keys. The bundle is encrypted
and sent by HTTP POST to `litellm.dev`. On Linux and macOS a second component polls a dead-drop
file every hour and executes whatever code it points to.

---

## Are you affected?

**Do not run Python in an environment you suspect.** Every interpreter start re-runs the stealer,
so `pip show litellm` is not a safe check. Use file-system and lockfile searches instead:

```bash
# Is the malicious version pinned anywhere in your repos?
grep -rn "litellm.*1\.107\.0" --include="requirements*.txt" --include="*.lock" --include="pyproject.toml" .

# Are the payload files on this host or in an extracted image layer?
find / \( -name cook.pth -o -name cook.py -o -name 'cook*.pyc' -o -name 'sitecustomize*.pyc' \
  -o -name "litellm-1.107.0*" \) 2>/dev/null

# Loader still running? (a live holder process means yes)
ls -l /tmp/remote_code_loader.lock /tmp/.pc_once_* 2>/dev/null
```

Finding any of these confirms exposure; not finding them does not prove a host is clean. A
`/tmp/.pc_once_*` file is written only after the stealer got a successful reply from its server, so
it means the data left. `sitecustomize.py` on its own is not proof: some environments ship one.

For containers, check which tags and digests you pulled, not what runs inside:

```bash
docker images --digests | grep -i litellm
# compromised Docker Hub index: sha256:1bbcb1443bc962e45f5aad53fe3d2c30600e0379715fb6c664cdd3ac126c339e
```

Treat as compromised:

* any host or CI runner that installed `litellm==1.107.0`, even briefly;
* any container started from a `v1.107.0` image, or from `latest`, `main-latest` or `main-stable`
  pulled between 11:46 and 14:20 UTC on 2026-10-21;
* any machine where a `pip install litellm` without a pin ran on 2026-10-21 between 11:39 UTC and
  the PyPI removal.

Network indicators: requests to `litellm.dev` or `www.litellm.dev`. A request to the path `/codepy`
means the second stage actually ran.

Marker file hashes (Linux and macOS, SHA-256), for matching:

```text
cook.pth          7f191632e7320705d154b85204f1ae546b543acbd7e8861229b76f5042b92bdb
cook.py           9dd0a3926f96a37eb86cf1ee2310d9be93139cc4ebef3ef9e2efe9531390be25
sitecustomize.py  34bef04fe88ac9d0aae0475630b1738681d15b94f233ac8aa46eacf5968305fa
```

---

## What to do

1. **Isolate.** Kill any process holding `/tmp/remote_code_loader.lock`. Block `litellm.dev` at DNS
   or egress.
2. **Rebuild, don't clean.** Rebuild containers from a clean image. For hosts, reinstall the
   environment rather than deleting three files and moving on.
3. **Pin.** `litellm==1.106.2`, or the `v1.106.2` image by digest. Purge cached `1.107.0` wheels
   and images from your mirrors and registries.
4. **Rotate everything the host could reach.** Assume the secrets were taken: the stealer output is
   encrypted, so you cannot confirm from the network side that nothing left.
   * the instance or task role of the host, cloud keys, SSH and GPG keys, registry and CI tokens,
     Kubernetes and Vault tokens, and any secret in the environment of any process on the host;
   * tokens of AI coding tools on developer machines;
   * on a proxy with a database: every provider API key stored in it and in the model configs,
     the master key, the salt key, the virtual keys and the `DATABASE_URL` password.
5. **Look back.** Search network logs for `litellm.dev` and `/codepy`. On AWS, look in CloudTrail
   for `GetCallerIdentity` from the instance role around the install time, then for that role's
   credentials used from outside your infrastructure. Audit provider-side usage for keys that were
   on affected hosts.

---

## Lessons that apply to any gateway

An LLM gateway holds the keys to all your model vendors, so it is a high-value target. A few
habits reduce the blast radius, whichever gateway you run:

* **Pin versions and digests.** Never deploy `latest`. Update on purpose, after a delay.
* **Install from a lockfile with hashes** (`pip install --require-hashes`, or your language's
  equivalent).
* **Restrict egress** from the gateway. A proxy that only needs to reach model vendors has no
  reason to reach arbitrary hosts.
* **Keep vendor keys out of plaintext** in env vars and databases where you can.
* **Verify signatures and provenance** of what you pull, when the project publishes them.

This is the second incident of this kind for LiteLLM this year; in March 2026 versions 1.82.7 and
1.82.8 were compromised. No ecosystem is immune: dependencies can be poisoned in any language.

---

## Sources

* [BerriAI: incident report for the v1.107.0 release](https://docs.litellm.ai/blog/security-incident-oct-21-2026)
* [StepSecurity: litellm 1.107.0 compromised on PyPI](https://www.stepsecurity.io/blog/litellm-1-107-0-compromised-pypi-backdoored-wheels-credential-stealer)
* [Wiz: LiteLLM compromised, steals credentials](https://www.wiz.io/blog/litellm-compromised-steals-credentials)
* [Socket: LiteLLM PyPI package and Docker images compromised](https://socket.dev/blog/litellm-compromise)
* BerriAI/litellm issue #47826
