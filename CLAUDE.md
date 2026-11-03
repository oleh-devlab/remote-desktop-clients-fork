# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A fork of iiordanov's remote-desktop-clients: one Android codebase that produces several apps — bVNC (VNC), aRDP (RDP), aSPICE (SPICE), Opaque (oVirt/RHEV/Proxmox over SPICE), their `free*` variants, and "Custom" VNC clients. Java + Kotlin on the app side, with heavy native dependencies (FreeRDP, GStreamer/spice-gtk, OpenSSL, libgovirt, etc.) built via the NDK.

## Building

Gradle will not even configure until native dependencies exist: `settings.gradle` includes `:remoteClientLib:jni:libs:deps:FreeRDP:client:Android:Studio:freeRDPCore`, which is only present after the prebuilt deps are downloaded or built from scratch. The sqlcipher AAR in `common/aars/` is also fetched by `prepare_project.sh`.

Fastest setup (prebuilt deps, then fix up libs per client without rebuilding native code):

```bash
./download-prebuilt-dependencies.sh
./bVNC/prepare_project.sh --skip-build libs nopath
```

From-scratch native build (takes hours; installs the NDK automatically; `docker/docker-compose.yml` wraps the same thing):

```bash
./bVNC/prepare_project.sh <PROJECT> $ANDROID_SDK
```

`PROJECT` is one of `bVNC freebVNC aRDP freeaRDP aSPICE freeaSPICE Opaque libs remoteClientLib` or `Custom*Vnc*`. **`prepare_project.sh` is stateful**: depending on the target it deletes/keeps `.so` files in `bVNC/libs` and `remoteClientLib/src/main/jniLibs`, renames FreeRDP's `jniLibs` ↔ `jniLibs.DISABLED`, and for Custom targets rewrites `CUSTOM_VNC_APP_NAMESPACE` in `gradle.properties`. Re-run it when switching between VNC / RDP / SPICE targets, and don't commit the `gradle.properties` change it makes.

Gradle (after deps are in place):

```bash
./gradlew :bVNC-app:assembleDebug        # or :aRDP-app, :aSPICE-app, :Opaque-app, :freebVNC-app, :CustomVnc-app ...
./gradlew :common:testDebugUnitTest      # the only unit tests (template-level); run one with --tests 'com.morpheusly.common.ExampleUnitTest'
./gradlew :bVNC:lint
```

Global SDK levels live in root `build.gradle` `ext` (`compileApi`/`targetApi` 36, Java 21); `minSdkVersion` comes from `SDK_VERSION` in `gradle.properties`. Native dependency versions, NDK versions and download URLs/hashes live in `remoteClientLib/jni/libs/build-deps.conf`; FreeRDP/cerbero/spice-gtk patches are the numbered `*.patch` files next to it and are applied by `build-deps.sh`.

## Architecture

- **`*-app` modules are thin shells.** Each just sets an application `namespace`/package and depends on `:bVNC`. All UI and protocol logic lives in the `:bVNC` library module (namespace `com.undatech.remoteClientUi`).
- **Client type is derived from the package name at runtime.** `Utils.isVnc/isRdp/isSpice/isOpaque/isFree/isCustom` (`bVNC/src/main/java/com/iiordanov/bVNC/Utils.java`) check whether the package name contains `vnc`/`rdp`/`spice`/`opaque`/`free`. Package naming therefore controls behaviour — keep it in mind when touching namespaces or adding clients.
- **Connection flow:** `ConnectionListActivity` / per-client launcher activities (`bVNC.java`, `aRDP.java`, `aSPICE.java`, packages `com.iiordanov.{bVNC,aRDP,aSPICE,free*}`) → `RemoteCanvasActivity` + `RemoteCanvas` (the drawing surface) → `protocol/RemoteConnectionFactory.kt` picks a `RemoteConnection` subclass (`RemoteVncConnection`, `RemoteRdpConnection`, `RemoteSpiceConnection`, `RemoteOvirtConnection`, `RemoteProxmoxConnection`, `RemoteOpaqueConnection`).
- **Protocol backends:** VNC is pure Java in `:bVNC` (`RfbProto`, decoders, `com.iiordanov.tigervnc`, SSH tunnelling via the vendored `com.iiordanov.jcraft`, `X509Tunnel`/`TLSTunnel`/`SecureTunnel`). RDP and SPICE go through `:remoteClientLib` (`RdpCommunicator` → FreeRDP's `freeRDPCore` module; `SpiceCommunicator` → JNI code in `remoteClientLib/jni/android` + `virt-viewer`, built with `ndk-build` from `jni/Android.mk`). Shared interfaces (`Connection`, `Viewable`, `RfbConnectable`, `InputCarriable`) live in `remoteClientLib/src/main/java/com/undatech/opaque`.
- **Input:** `bVNC/.../input/` — `TouchInputHandler*` variants implement the touch modes; `RemoteKeyboard`/`RemotePointer` have per-protocol subclasses (`RemoteVnc*`, `RemoteRdp*`, `RemoteSpice*`).
- **Storage:** connections are stored in an SQLCipher-encrypted database (`Database.java`, `ConnectionBean`/`AbstractConnectionBean`; `:common` holds shared DB helpers).
- **Custom VNC clients:** a YAML file at `bVNC/src/main/assets/com.iiordanov.<Name>.yaml` (template: `custom_vnc_client.yaml-EXAMPLE`) hides/locks UI fields; `CUSTOM_VNC_APP_NAME`/`CUSTOM_VNC_APP_ICON` in `gradle.properties` feed `CustomVnc-app` manifest placeholders. See README §III.
- Changelogs per app: `bVNC/CHANGELOG-*`. Custom CAs for aSPICE/Opaque go in `remoteClientLib/certificate_authorities/` and are appended to the bundled `ca-certificates.crt` at build time.

## Outdated / security-sensitive areas — be careful

The maintainer flagged that parts of this repo are outdated and vulnerable. Known hotspots:

- **Native deps are old:** OpenSSL `1.1.1w` (EOL), FreeRDP `2.11.7` (2.x line), gmp/nettle/libsoup 2.x versions in `build-deps.conf`. Bumping them usually requires reworking the numbered patches and is a multi-hour native rebuild — don't bump casually, and don't assume a version bump is a drop-in fix.
- **Unverified / plaintext downloads:** several URLs in `build-deps.conf` use `http://` (GStreamer, config.guess/sub, spice-space); `download-prebuilt-dependencies.sh` fetches a tarball with no checksum; `prepare_project.sh` downloads the NDK and sqlcipher AAR without verification. Prefer adding hashes/HTTPS over adding new unverified fetches.
- **Old Java deps:** `jcifs 1.3.17` (SMBv1-era), `httpcore 4.4.10`, legacy `com.android.support:*` alongside AndroidX, and `remoteClientLib/build.gradle` still declares the dead `jcenter()` repo in its own `buildscript`.
- **Certificate handling is deliberate:** `X509Tunnel`/`SecureTunnel` use custom `X509TrustManager`s that surface unknown certs to the user for trust-on-first-use, and `03_freerdp_remove_cert_ignore.patch` removes FreeRDP's cert-ignore path. Don't replace these with accept-all trust managers or re-enable cert ignoring.
- `docker/Dockerfile` grants the build user passwordless sudo and enables `protocol.file.allow always` — fine for a throwaway build container, not something to copy elsewhere. `.env` at the repo root is a symlink to `docker/.env`.
