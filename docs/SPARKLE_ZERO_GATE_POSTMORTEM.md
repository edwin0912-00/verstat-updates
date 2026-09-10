# Sparkle Update Gate Postmortem: Resolution of Repeated Full Downloads

**Date:** 2026-09-10  
**Affected Components:** Auto-Update Feed (`astra-dev-alpha/appcast.xml`), Sparkle 2 Client Update Engine  
**Target Public Key:** `5cJaL+sarJ8qqHI/UB5pObvqtbrqwuKDhsedlXS63yk=` (`variator_astra_dev_alpha`)  
**Resolution Commit:** `934c021`  
**Status:** RESOLVED & AUTOMATED GATE ENFORCED

---

## 1. Executive Summary

During recent auto-update cycles across the fleet (specifically targeting transitions from Build 326 through Build 343 and 345), client machines repeatedly downloaded the full release zip archive (~477 MB to 1.6 GB) instead of downloading and applying the intended lightweight binary delta updates (~500 KB to 2 MB).

This occurred despite:
- Valid binary deltas being built and verified locally (`BinaryDelta apply`).
- Deltas being signed with the proper Keychain key (`variator_astra_dev_alpha`).
- Deltas being uploaded and available on GitHub Releases.
- Enclosure URLs and cryptographic signatures being listed in `appcast.xml`.

The issue resulted in excessive network bandwidth usage, prolonged update times across fleet MacBooks, and unnecessary client strain.

---

## 2. Root Cause Analysis: XML Hierarchy Mismatch

The root cause was an **invalid XML hierarchy** in `astra-dev-alpha/appcast.xml`. Specifically, the `<sparkle:deltas>` element was erroneously nested **inside** the primary release `<enclosure>` tag, instead of being placed as a direct child of `<item>` (sibling to `<enclosure>`).

### Buggy Structure (Root Cause)

```xml
<!-- ❌ INVALID: sparkle:deltas nested INSIDE enclosure -->
<item>
  <title>Version 0.3.0 (345)</title>
  <pubDate>Thu, 10 Sep 2026 17:24:56 +0000</pubDate>
  <sparkle:minimumSystemVersion>15.0</sparkle:minimumSystemVersion>
  <description><![CDATA[ ... ]]></description>
  <enclosure url="https://.../Variator-0.3.0-345.zip" sparkle:version="345" ...>
    <sparkle:deltas>
      <enclosure url="https://.../Variator345-343.delta" sparkle:deltaFrom="343" ... />
    </sparkle:deltas>
  </enclosure>
</item>
```

### Why This Broke Sparkle 2

1. **RSS 2.0 Enclosure Semantics:** In the RSS 2.0 specification, `<enclosure>` is defined as an empty/leaf XML element carrying media attributes (`url`, `length`, `type`). It has no child elements.
2. **Sparkle 2 Appcast Parser Behavior (`SUAppcastItem`):**
   - Sparkle's appcast parser searches for `<sparkle:deltas>` directly under the `<item>` tag.
   - When the parser processes `<enclosure>`, it extracts its attributes and closes the element. It does not recurse into `<enclosure>` to look for delta updates.
3. **Silent Delta Dropout:**
   - Because the deltas were nested inside `<enclosure>`, Sparkle's parser found zero delta items (`item.deltas.count == 0`).
   - When a client running e.g. Build 343 or Build 326 queried the feed, Sparkle checked if any delta matched the installed version.
   - Finding no deltas registered on the item, Sparkle silently fell back to its default behavior: **downloading the full archive from the main `<enclosure>`.**

---

## 3. The Resolution

### Correct XML Structure

The XML hierarchy was corrected across all release items in `astra-dev-alpha/appcast.xml` so that:
- `<enclosure>` is self-closing (`<enclosure ... />`).
- `<sparkle:deltas>` is a direct child of `<item>`, placed as a sibling to `<enclosure>`.

```xml
<!-- ✅ CORRECT: enclosure is self-closing, sparkle:deltas is a direct sibling under item -->
<item>
  <title>Version 0.3.0 (345)</title>
  <pubDate>Thu, 10 Sep 2026 17:24:56 +0000</pubDate>
  <sparkle:minimumSystemVersion>15.0</sparkle:minimumSystemVersion>
  <description><![CDATA[ ... ]]></description>
  <enclosure url="https://github.com/edwin0912-00/verstat-updates/releases/download/astra-v0.3.0-345/Variator-0.3.0-345.zip" sparkle:version="345" sparkle:shortVersionString="0.3.0" length="1609296725" type="application/octet-stream" sparkle:edSignature="7SOodIKiqV9VbitVxqxoE1Vr36alBEn4euO+I0RQKrNGGHo3eITxh++CWLcArK1pO7PAtLEgFn/7lMnq4/c7Dw==" />
  <sparkle:deltas>
    <enclosure url="https://github.com/edwin0912-00/verstat-updates/releases/download/astra-v0.3.0-345/Variator345-343.delta" sparkle:version="345" sparkle:shortVersionString="0.3.0" sparkle:deltaFrom="343" length="574046" type="application/octet-stream" sparkle:edSignature="/TmpxxbizONm2HKS+fJmVORbiriHWz8PwZntnS6lz0vsg4LI2Mnnkapek5xEi0/sA8WNGL0brz3Lwqrjl7j2BQ==" />
  </sparkle:deltas>
</item>
```

### Feed Re-Signing

After restructuring the XML, the feed was re-signed using the project's dedicated Keychain account:

```bash
sign_update --account variator_astra_dev_alpha astra-dev-alpha/appcast.xml
```

This updated the cryptographic `<!-- sparkle-signatures: -->` block at the end of the file.

---

## 4. Mandatory Invariants for All Agents & Pipelines

Every autonomous agent, workflow, or engineer modifying Sparkle feeds or publishing releases MUST adhere strictly to the following 5 invariants:

### Invariant 1: Sibling Tag Hierarchy (Zero Enclosure Nesting)
- The primary `<enclosure>` element MUST be empty and self-closing (`<enclosure ... />`).
- `<sparkle:deltas>` MUST be a direct child of `<item>`.
- **NEVER** nest `<sparkle:deltas>` inside `<enclosure>`.

### Invariant 2: No `<sparkle:channel>` on Public / General Feeds
- In Sparkle 2, any `<item>` containing `<sparkle:channel>` is discarded by clients unless their host app delegate implements `allowedChannelsForUpdater:` and explicitly whitelists the channel.
- Releases intended for general clients or the standard test fleet must completely **omit** the `<sparkle:channel>` element.

### Invariant 3: Strict Keychain Key Specification (`--account`)
- **NEVER** run bare `sign_update` without `--account`.
- Always execute:
  ```bash
  sign_update --account variator_astra_dev_alpha <target_file>
  ```
- Bare `sign_update` silently signs with the default `ed25519` key, which does not match `SUPublicEDKey` (`5cJaL+sarJ8qqHI/UB5pObvqtbrqwuKDhsedlXS63yk=`), causing client verification rejection (*"The update is improperly signed and could not be validated"*).
- Both the enclosure files (`.zip`, `.delta`) AND the `appcast.xml` feed MUST be signed with this exact account.

### Invariant 4: Standalone Full Bundle for Delta Generation
- Deltas must **only** be generated from full, standalone release packages (`package_apple_silicon.sh`, ~1.67 GB) containing the complete bundled runtime (`runtime/bin/node` >= 50MB, `ffmpeg`, `ffprobe`, `runtime/lib/`, `runtime/project/`, `package-runtime.json`).
- **NEVER** generate deltas from development builds (`dist/development`, `development-runtime.json`). Doing so causes `BinaryDelta` to instruct clients to delete 38,000+ files and strips the runtime from the app.

### Invariant 5: Mandatory Automated Preflight Gate
- Before committing or pushing any change to an appcast feed, run the automated verification script:
  ```bash
  ./script/verify_sparkle_feed.sh path/to/appcast.xml
  ```
- All 4 checks must return `PASS`:
  - **Check 1:** Zero `<sparkle:channel>` tags.
  - **Check 2:** Cryptographic Ed25519 signature validation of feed against `SUPublicEDKey`.
  - **Check 3:** Structural validation ensuring `<sparkle:deltas>` is a direct sibling of `<enclosure>` and `<enclosure>` has zero child elements.
  - **Check 4:** Cryptographic validation of every enclosure and delta URL against `SUPublicEDKey`.

---

## 5. Summary Reference

| Property | Correct Setting | Incorrect Setting (Defect) |
|---|---|---|
| `<enclosure>` | Self-closing: `<enclosure ... />` | Block wrapper: `<enclosure>...</enclosure>` |
| `<sparkle:deltas>` | Direct child of `<item>` (sibling to `<enclosure>`) | Child of `<enclosure>` |
| `<sparkle:channel>` | Omitted on general/dev-alpha feeds | `<sparkle:channel>0.3</sparkle:channel>` (causes update blindness) |
| Signing Command | `sign_update --account variator_astra_dev_alpha ...` | `sign_update ...` (wrong default key) |
| Preflight Gate | `./script/verify_sparkle_feed.sh <appcast.xml>` | Manual inspection or unverified push |
