---
layout: post
title: "Verifying a Keystone release: cosign, SBOM and SLSA provenance"
date: 2026-11-01 07:00:00 +0000
reading_time: 6
tags:
  - security
  - supply-chain
  - ktms
  - cosign
description: "A practical guide to checking that the Keystone image or binary you run is the one our release workflow built: keyless cosign signatures, SBOMs, SLSA provenance, and what a signature does not guarantee."
---

An LLM gateway holds the API keys of every model vendor you use. That makes "is this really the
image I think it is?" a more pressing question for a gateway than for most dependencies.

Every Keystone. The Model Switchboard. (KTMS) release since 0.14.0, including all of 1.x, ships with
three things you can check yourself: signatures, software bills of materials and build provenance.
This post shows how to check each one, and what each one does and doesn't prove.

---

## What is published

| Artifact | What comes with it |
|---|---|
| Docker images `coyl/ktms` (every platform manifest) | Keyless cosign signature, SBOM attestation, SLSA provenance (`mode=max`) |
| Release archives `ktms-server_*`, `ktms-node_*` | An SPDX SBOM per archive (`*.sbom.json`) |
| `checksums.txt` of the GitHub Release | Keyless cosign signature as a Sigstore bundle (`checksums.txt.sigstore.json`) |

**Keyless** means there is no long-lived signing key to steal. The release workflow gets a
short-lived certificate from Sigstore for its GitHub Actions OIDC identity, signs, and the signature
is recorded in a public transparency log. The identity in the certificate is the workflow file and
the tag it ran for.

## Verify an image

You need cosign 3.

```bash
V=v1.0.0
cosign verify coyl/ktms:${V#v} \
  --certificate-identity "https://github.com/ai-process/ktms/.github/workflows/docker-publish.yml@refs/tags/$V" \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

Pin the identity to the **exact tag** you're verifying. A looser pattern such as "any workflow in
this repository" would also accept a signature made by a build of some other branch.

Better still, deploy by digest and verify the digest:

```bash
DIGEST=$(docker buildx imagetools inspect coyl/ktms:${V#v} --format '{{ json .Manifest.Digest }}' | tr -d '"')
cosign verify "coyl/ktms@$DIGEST" \
  --certificate-identity "https://github.com/ai-process/ktms/.github/workflows/docker-publish.yml@refs/tags/$V" \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

A tag can be moved to another image; a digest cannot.

## Read the SBOM and provenance

```bash
docker buildx imagetools inspect coyl/ktms:${V#v} --format '{{ json .SBOM }}'
docker buildx imagetools inspect coyl/ktms:${V#v} --format '{{ json .Provenance }}'
```

The **SBOM** lists every package in the image, so your scanner (Trivy, Grype, or whatever you use)
can match it against vulnerability databases without pulling and unpacking the image.

The **provenance** records how the image was built: the source repository and commit, the workflow,
the build parameters. `mode=max` includes the full build definition. Use it to answer "which commit
is this?" without trusting the tag.

## Verify a binary

```bash
cosign verify-blob checksums.txt --bundle checksums.txt.sigstore.json \
  --certificate-identity "https://github.com/ai-process/ktms/.github/workflows/release.yml@refs/tags/$V" \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
sha256sum --ignore-missing -c checksums.txt
```

The first command proves `checksums.txt` came from our release workflow for that tag. The second
proves your downloaded archives match it.

## What else runs before a release

Signatures say who built something. These gates decide whether it gets built at all:

* **`govulncheck`** for linux and darwin on every pull request and before every release. A known
  vulnerability that KTMS code can actually reach fails the build.
* **`buf breaking`** against the previous release, so the 1.x API doesn't break by accident.
* **An upgrade test**: the previous release and the oldest release of the same major write a
  database, the new build migrates and serves it, and the old release runs again on the migrated
  database.
* **Ordering**: binaries are published only after those checks pass, and images only after the
  binaries.
* **Pinned tools**: the actions in the signing jobs are pinned by commit, and the Go and Alpine
  versions are fixed.

## What a signature does not prove

This is the part most guides skip.

* **A signature proves origin, not safety.** It says "our release workflow built this from that
  tag". If someone gained the rights to change our release workflow or push a tag, they could
  produce a validly signed malicious image. Pinning the identity to the tag narrows this, but the
  real protection is who can change the workflow, which is access control, not cryptography.
* **An SBOM is a list, not a verdict.** It tells your scanner what to check. It doesn't say
  anything is safe.
* **Provenance describes the build, not the code.** A build of a bad commit has perfectly good
  provenance.
* **None of this protects a running host.** Keystone encrypts vendor keys in Postgres, but the
  keyring is in the process environment. Anything that can read that environment can read the
  keys.

## A short checklist

1. Deploy by digest, not by tag.
2. Verify the signature with the identity pinned to the exact tag.
3. Feed the SBOM to your scanner.
4. Don't upgrade on release day; a short delay costs little and catches a lot.
5. Restrict the gateway's egress to your model vendors.
6. Keep the keyring in a secret manager, apart from database backups.

* Security policy and the same commands: [`SECURITY.md`](https://github.com/ai-process/ktms/blob/main/SECURITY.md#verifying-a-release)
* Releases: [github.com/ai-process/ktms/releases](https://github.com/ai-process/ktms/releases)
