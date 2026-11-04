# P2 — Android 17 e2fsprogs, fscrypt metadata and recovery compatibility

**Date:** 2026-11-04. **Scope:** preserve historical PE intent, compare platform owners, avoid silent filesystem validation regressions. This is not a filesystem repair test.

## Source owners

* Historical PE `seventeen` SHA `d9cfad90b6a824594d14aa4c05d81f69b489ea58` (2023); its last change is [loosen fscrypt policy xattr size](https://github.com/PixelExperience-Reborn/external_e2fsprogs/commit/d9cfad90b6a824594d14aa4c05d81f69b489ea58).
* Android17 official `platform/external/e2fsprogs` tag `android-17.0.0_r1`: commit `3eb17241db7cd4408cc1bce9d25b602465ace8ff`, tree `804f47c7a1bc432556ff3b39bf2d47513ba630e0`; METADATA identifies upstream e2fsprogs 1.47.2. Source: https://android.googlesource.com/platform/external/e2fsprogs/+/refs/tags/android-17.0.0_r1
* Active staged AOSP17 manifest already owns `external/e2fsprogs` with **`platform/external/e2fsprogs`**, not the 2023 PE fork.
* `vendor/aosp/config/common.mk` selects `mke2fs`, so **the official source is a real platform dependency**, not merely an archival library. A vendor source-owner preflight on the reviewed vendor HEAD is successful: https://github.com/PixelExperience-Reborn/vendor_aosp/actions/runs/41863699282. This does not establish device filesystem correctness.

## Critical historical change: fscrypt xattr length

PE fork `e2fsck/encrypted_files.c` relaxed the size condition for **both** fscrypt context v1/v2:

```diff
- if (xattr_size != sizeof(*ctx))
+ if (xattr_size < sizeof(*ctx))
```

The original intent was to accept extended OEM kernel xattrs. The behavioral effect is to accept **extra trailing bytes** as well as canonical contexts. This must not be carried automatically into a generic Android17 ext4 recovery tool: the relaxed check changes corruption detection and could accept malformed or incompatible metadata.

**Decision:** retain the official strict fscrypt size verification. Revisit only with an exact device-specific xattr specimen, documented OEM fscrypt extension and its kernel semantics, test vectors covering truncated/exact/oversized/corrupted inputs, and review of the e2fsprogs upstream fix. Never apply an unconditional policy relaxation at the platform level.

## Build/recovery boundary

Historical PE also changed `e2fsck`, `resize2fs`, `tune2fs` recovery availability and added `resize2fs_static`. Compare these **against official Android17 Soong modules** under `e2fsck/`, `misc/`, `resize/` on the actual pinned tree before selecting any recovery-only modification. Do not assume a 2023 module declaration is still compatible. The official Android17 `Android.bp` includes the `e2fsprogs-defaults` module and subdirectory hierarchy.

## Next decisive checks

1. Android17 `mke2fs` and `e2fsck` selected Soong targets compile, including their exact recovery variants if required by PE recovery.
2. Host image fixtures: ext4 feature flags, fscrypt v1/v2 exact/short/oversize xattrs, metadata checks, and fsck read-only dry run. Never test with a real userdata partition until correctness is established.
3. Only then evaluate whether an isolated compatibility path is needed. Device-specific concerns belong at the later device enablement stage, not in the base manifest.

**Completed:** source ownership, historical patch semantics, official AOSP17 baseline metadata, vendor source-owner CI. **Not completed:** full Soong build, fsck repair or encryption recovery, kernel interoperability and device testing.
