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

This is a developing incident. Details below come from public write-ups published within hours of
the release and may change. Check the sources at the end and the BerriAI advisory before acting.

---

## What happened

Times are UTC, 2026-10-21, as reported by [StepSecurity](https://www.stepsecurity.io/blog/litellm-1-107-0-compromised-pypi-backdoored-wheels-credential-stealer):

| Time | Event |
|---|---|
| 08:56–10:46 | Commits pushed to a lookalike repository, `Berri-AI/litellm` (the real one is `BerriAI/litellm`) |
| 11:39 | `litellm==1.107.0` uploaded to PyPI |
| 11:48 | Infected image pushed to Docker Hub; floating tags point to it |
| 12:06 | Compromise reported in BerriAI/litellm issue #47826 |
| by 13:23 | `1.107.0` removed from PyPI |
| 14:32 | BerriAI confirms the infected Docker images are removed |

How the release was published is still unresolved: BerriAI is investigating.

### What is affected

* **PyPI:** `litellm==1.107.0`, all seven wheels (macOS, Linux glibc and musl, Windows). The sdist
  has a tampered build config but no payload.
* **Docker:** the `v1.107.0` tag of `ghcr.io/berriai/litellm`, `litellm-database`,
  `litellm-non_root` and `docker.io/litellm/litellm`. The floating tags `latest`, `main-latest` and
  `main-stable` pointed to the infected image for about three hours.
* **Reported clean:** `1.107.0rc1`, `1.107.0rc2` and the `1.103.x`–`1.106.x` releases.

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
find / \( -name cook.pth -o -name cook.py -o -name "litellm-1.107.0*" \) 2>/dev/null

# Loader still running? (a live holder process means yes)
ls -l /tmp/remote_code_loader.lock /tmp/.pc_once_* 2>/dev/null
```

For containers, check which tags and digests you pulled, not what runs inside:

```bash
docker images --digests | grep -i litellm
```

Treat as compromised:

* any host or CI runner that installed `litellm==1.107.0`, even briefly;
* any container started from a `v1.107.0` image, or from `latest`, `main-latest` or `main-stable`
  pulled during the roughly three-hour window on 2026-10-21;
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
   * cloud keys, SSH and GPG keys, registry and CI tokens, Kubernetes and Vault tokens;
   * tokens of AI coding tools on developer machines;
   * on a proxy with a database: every provider API key stored in it and in the model configs,
     plus the master key and the salt key.
5. **Look back.** Search network logs for `litellm.dev` and `/codepy`, and audit provider-side
   usage for keys that were on affected hosts.

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

* [StepSecurity: litellm 1.107.0 compromised on PyPI](https://www.stepsecurity.io/blog/litellm-1-107-0-compromised-pypi-backdoored-wheels-credential-stealer)
* [Wiz: LiteLLM compromised, steals credentials](https://www.wiz.io/blog/litellm-compromised-steals-credentials)
* [Socket: LiteLLM PyPI package and Docker images compromised](https://socket.dev/blog/litellm-compromise)
* BerriAI/litellm issue #47826
