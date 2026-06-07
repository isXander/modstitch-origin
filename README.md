# Modstitch Origin

**Modstitch Origin** is a draft specification for helping Minecraft mod authors prove that release artifacts came from their own release process.

* It is not a malware scanner.  
* It is not a trust badge.  
* It does not prove that a mod is safe.

It answers one narrower question:

> Did this downloaded artifact satisfy the author’s declared origin policy?

The draft RFC is currently being discussed here:

[Discuss RFC 0001: Modstitch Origin Manifest v1.0](https://github.com/isXander/modstitch-origin/issues/1)

Read the current draft RFC here:

[RFC 0001: Modstitch Origin Manifest v1.0](https://modstitch-origin.isxander.dev/rfc/0001)

## Why this exists

Minecraft mods have been hit by supply-chain attacks before. One serious attack vector is the theft of publishing credentials for platforms like Modrinth or CurseForge.

If an attacker steals a publishing token, they may be able to upload a poisoned JAR directly to a legitimate project page. To users and launchers, the file may appear to come from the correct project, on the correct platform, from the correct author. Clients such as launchers may also facilitate one-click updates that bypass the need for manual verification.

Ideally, mod platforms would solve this themselves by supporting first-class artifact signing requirements:

- authors enable signing or attestation for a project;
- authors upload trusted public keys or configure trusted CI attestations;
- the platform rejects uploads that do not satisfy the project’s policy;
- publishing tokens cannot disable or weaken that policy;
- disabling signing requires re-authentication or second-factor confirmation;
- large projects may be required to use signing.

That is the best long-term solution.

But such security measures are not being taken seriously by mod platforms, and attackers continue to utilize supply-chain attacks to spread malware.

Modstitch Origin is an attempt to let authors and third-party launchers to take some of that protection into their own hands today.

## How it works

A mod author publishes a JSON manifest, conventionally named:

```text
.modstitch-origin.json
````

The manifest declares:

* which distribution sources it applies to;
* which Modrinth or CurseForge project IDs are covered;
* from what date verification should apply;
* which signatures or attestations can prove that an artifact came from the author’s release process.

A launcher, mod manager, or other client can then check downloaded artifacts against that manifest.

For example, a project may say:

> Releases after 2026-07-01 must either be signed by this OpenPGP key or have a GitHub artifact attestation from this repository.

If a downloaded file does not satisfy that policy, the client can warn, block, log, or ask the user what to do.

The RFC deliberately does **not** require a particular enforcement behaviour.

## What “origin verified” means

If an artifact passes Modstitch Origin verification, it means:

> This artifact satisfied the project’s declared origin policy.

It does **not** mean:

* the mod is safe;
* the mod is open source;
* the mod is malware-free;
* the build is reproducible;
* the code has been audited;

A malicious author can still sign their own malware.

This is about protecting against a different class of attack: someone other than the author uploading a file that does not match the author’s declared release process.

## Minimal example

```json
{
  "schemaVersion": 1,
  "enforceAfter": "2026-07-01T00:00:00Z",

  "sources": [
    {
      "type": "modrinth",
      "projectId": "AANobbMI",
      "validFrom": "2026-07-01T00:00:00Z",
      "validUntil": null
    }
  ],

  "verification": {
    "mode": "anyOf",
    "tags": [
      "release-key"
    ]
  },

  "verifiers": [
    {
      "id": "release-key",
      "type": "openpgp-signature",
      "validFrom": "2026-07-01T00:00:00Z",
      "validUntil": null,
      "fingerprint": "0123456789ABCDEF0123456789ABCDEF01234567",
      "keyservers": [
        "hkps://keyserver.ubuntu.com"
      ]
    }
  ]
}
```

In this example, releases from the listed Modrinth project after `2026-07-01T00:00:00Z` are expected to have a detached OpenPGP signature from the declared key.

For an artifact named:

```text
example.jar
```

clients look for sidecar signatures such as:

```text
example.jar.asc
example.jar.sig
```

## For mod authors

Modstitch Origin is meant to be easy to adopt.

At its simplest, you can:

1. Create an OpenPGP release signing key.
2. Upload the public key to a keyserver, or embed it in the manifest.
3. Sign your release JARs during publishing.
4. Upload the detached `.asc` or `.sig` sidecar files alongside your artifacts.
5. Add `.modstitch-origin.json` to your repository.
6. Make sure your Modrinth/CurseForge project links to that repository.

If you already publish through GitHub Actions, you may be able to use GitHub artifact attestations instead of, or in addition to, OpenPGP signatures.

Important operational advice:

* keep release signing keys secure;
* do not store signing keys somewhere that defeats the point of requiring multiple verifiers;
* scope publishing tokens narrowly;
* do not give CI tokens permission to edit project metadata;
* monitor changes to `.modstitch-origin.json`;
* use branch protection or CODEOWNERS for the manifest.

A publishing token used by CI should ideally be able to upload files, but not:

* remove the project source URL;
* remove a dedicated Modstitch manifest URL;
* transfer project ownership;
* change team members;
* disable security settings;
* edit other security-sensitive metadata.

## For launcher and mod manager developers

You do not need to treat Modstitch Origin as mandatory enforcement.

The RFC is mostly about three things:

1. Finding a manifest.
2. Evaluating whether it applies to an artifact.
3. Reporting whether the artifact satisfies it.

Clients can decide their own UX.

Possible behaviours include:

* block failed artifacts;
* warn before installation;
* show a passive warning;
* record verification results in logs;
* expose verification results to modpack tooling;
* allow deliberate user bypass.

Recommended language:

```text
Origin verified
```

```text
Origin verification failed
```

```text
This artifact satisfies the project's Modstitch Origin policy.
```

```text
This artifact did not satisfy the author's declared release-origin requirements.
```

Avoid language like:

```text
safe
trusted
malware-free
clean
verified mod
approved by Modstitch
```

Those words imply safety or endorsement, which this spec does not provide.

### Important timestamp rule

All validity checks should use the timestamp provided by the distribution source.

Do not use timestamps from:

* JAR entries;
* ZIP entries;
* local filesystem metadata;
* artifact manifests;
* filenames;
* build metadata;
* signature creation time;
* attestation creation time.

Artifact-controlled timestamps can be spoofed or deliberately normalised for reproducible builds.

---

## For mod platforms

The best version of this idea belongs directly in mod platforms.

Client-side verification is useful, but platform-side enforcement is stronger.

A platform-native implementation could allow authors to:

* enable required artifact signing per project;
* upload trusted OpenPGP public keys;
* configure trusted GitHub/GitLab/Sigstore attestations;
* reject unsigned or unattested uploads;
* expose verification status through the API;
* expose a dedicated Modstitch manifest URL relation;
* prevent publishing tokens from disabling verification;
* require re-authentication or 2FA to weaken signing requirements;
* require signing for large or high-risk projects.

This RFC is not an argument against platform-side signing. It is a bridge for an ecosystem that does not yet have it everywhere.

If you work on a mod platform, first-class signing requirements would be better than making launchers reconstruct all of this client-side.

## For modpack maintainers

Modstitch Origin could eventually help modpack tools distinguish between:

* artifacts that satisfy an author’s origin policy;
* artifacts that fail an author’s origin policy;
* artifacts from projects that have not opted in.

This could be useful for:

* high-security modpacks;
* enterprise or educational deployments;
* public servers;
* packs that want stricter supply-chain hygiene;
* auditing downloaded artifacts.

The v1.0 draft does not yet define modpack lockfile integration, but it is listed as future work.

## Common concerns

### “Most mods will not opt in.”

Probably true, especially at first.

A missing manifest is not suspicious by itself. This is opt-in.

The goal is not to instantly protect every mod. The goal is to give authors, especially large or frequently targeted authors, a way to make poisoned platform uploads easier to detect.

### “Malware authors can sign their own malware.”

Yes.

This is not a malware reputation system. It does not decide whether a project is good or bad.

It only checks whether an artifact matches the release-origin policy declared by that project.

### “This should be implemented by Modrinth and CurseForge instead.”

Yes, ideally.

Platform-side signing enforcement is stronger and should happen.

Modstitch Origin exists because authors and launchers can implement part of the idea despite distribution platforms not taking security as seriously as they should.

### “If a stolen token can remove the source URL, this can be bypassed.”

Yes.

Version 1.0 does not solve downgrade attacks or source-link removal.

That is why authors should properly scope publishing tokens, and why platforms should separate publishing permissions from project configuration permissions.

Future versions may define client-side policy pinning, manifest history, signed manifests, or transparency logs. And discussion regarding these features is welcome.

### “What about locally compiled builds?”

Local development builds are out of scope.

This spec applies to artifacts obtained from declared distribution sources, such as Modrinth or CurseForge.

A local build can be legitimate without having a Modstitch Origin result.

## Current verifier types

The v1.0 draft currently defines two verifier types.

### `openpgp-signature`

Uses detached OpenPGP signatures such as:

```text
mod.jar.asc
mod.jar.sig
```

This works for both open-source and closed-source projects.

It proves that the artifact was signed by a key declared in the manifest.

### `github-attestation`

Uses GitHub artifact attestations.

This is useful for projects that build and publish release artifacts through GitHub Actions.

It can verify that an artifact digest is associated with a particular GitHub repository, workflow, and ref pattern.

## Current source types

The v1.0 draft currently defines:

```text
modrinth
curseforge
```

Future versions may add:

```text
github-release
gitlab-release
generic-url
```

## Future work

Potential future versions may cover:

* downgrade protection;
* client-side manifest pinning;
* signed manifests;
* manifest transparency logs;
* artifact-bound manifest references;
* GitHub Releases as a source;
* GitLab attestations;
* Codeberg or Forgejo support;
* Sigstore keyless signing;
* modpack lockfile integration;
* first-class platform upload enforcement.

## Getting involved

This is a draft.

The goal is to make something that is:

* useful for mod authors;
* realistic for launcher developers;
* simple enough to implement;
* honest about its limitations;
* compatible with future platform-side enforcement.

Discussion is happening here:

[Discuss RFC 0001: Modstitch Origin Manifest v1.0](https://github.com/isXander/modstitch-origin/issues/1)

Feedback is especially welcome from:

* mod authors;
* launcher developers;
* modpack maintainers;
* Modrinth/CurseForge platform developers;
* security-minded community members;
* people who have dealt with compromised publishing tokens;
* people who think this proposal is flawed.

Scepticism is useful. The point of the draft is to find the sharp edges before anyone treats it as a standard.
