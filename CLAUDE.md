# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this app is

Traccar Relay is a single-module Android app (`org.traccar.relay`) that bridges
Google's **Find My Device network** (FMDN / "Spot") to a self-hosted **Traccar**
server. It reverse-engineers the Google account, GCM/FCM checkin, and Nova
("Find My Device") protocols to request a tracker's location, receive the
encrypted location push, decrypt it client-side, and forward the resulting
lat/lon to a Traccar OsmAnd endpoint (`POST` form fields `id`, `lat`, `lon`,
`timestamp`, …). There is no first-party backend — Google's servers are the
backend.

## Build & run

```bash
./gradlew assembleDebug          # build debug APK
./gradlew assembleRelease        # build signed release (needs keystore, see below)
./gradlew installDebug           # install on connected device/emulator
./gradlew lint                   # Android lint
./gradlew clean
```

- No test source set exists yet. There are no unit/instrumented tests to run.
- Min SDK 26, target/compile SDK 37, Java 11, Kotlin + Jetpack Compose (Material3).
- Dependency versions live in `gradle/libs.versions.toml` (version catalog).
  Add libraries there, reference via `libs.*` in `app/build.gradle.kts`.
- **Release signing** reads `../environment/key.properties` (one directory *above*
  the repo root) for `keyAlias`/`keyPassword`/`storePassword`, with the keystore at
  `../environment/android.keystore`. If that file is absent the release build is
  unsigned — the build still configures, it just skips the signing config.
- `app/google-services.json` is required by the `google-services` plugin (Firebase
  Messaging). The Firebase project here is only used to obtain an FCM device token
  on the phone; the *tracker* push path uses hand-rolled GCM/MCS (see below).

## Protobuf (Wire)

`.proto` files in `app/src/main/proto/` are compiled by the **Wire** Gradle plugin
into Kotlin classes under the `org.traccar.relay.proto.*` packages (generated at
build time into `app/build/generated/`, not committed). The three schemas:

- `checkin.proto` → `org.traccar.relay.proto.checkin.*` — Android device checkin.
- `mcs.proto` → `org.traccar.relay.proto.mcs.*` — Mobile Connection Server (the
  GCM TCP push protocol stanzas).
- `device_update.proto` → `org.traccar.relay.proto.*` — Nova/FMDN device list,
  execute-action, and the encrypted location report payloads.

Decode/encode use Wire's generated `ADAPTER` and `.encode()` APIs. When you change
a `.proto`, rebuild before referencing new fields.

## Architecture & the core flow

The whole app orbits one orchestrator: **`api/DeviceRepository`** (god node, ~26
edges). Trace any feature through it. The end-to-end "relay a location" path:

1. **Login** (`auth/LoginScreen`) — a `WebView` loads Google's `EmbeddedSetup`
   page and polls cookies for an `oauth_token`. That master token + email go to
   `MainViewModel.onTokenReceived` → `DeviceRepository.saveCredentials`.
2. **Token exchange** (`auth/GoogleAuthClient`, against
   `android.clients.google.com/auth`):
   - `exchangeToken` trades the `oauth_token` for a long-lived **AAS token**.
   - `performOAuth` mints per-service tokens from the AAS token by scope:
     `android_device_manager` (the **ADM token** for Nova calls) and `spot`
     (for the owner-key fetch). All cached in `TokenStorage`.
3. **Shared-key setup** (`auth/KeySetupScreen`) — a second WebView flow yields the
   account's E2EE **shared key**; `fetchOwnerKey` then decrypts the per-account
   **owner key** via `SpotApiClient` + `LocationDecryptor.decryptOwnerKey`.
4. **FCM/GCM registration** (`push/FcmRegistrationClient.register`) — performs
   GCM checkin → c2dm register → Firebase installation → FCM register, producing
   an `FcmCredentials` bundle (gcm android id/security token, fcm token, and an
   EC keypair + auth secret for HTTP-ECE). Cached as JSON in `TokenStorage`.
5. **Device list** (`api/NovaApiClient.listDevices`) — Nova `nbe_list_devices`
   with the ADM token returns trackers as `Device(name, id)`.
6. **Request a location** — `NovaApiClient.buildLocationRequest` +
   `executeAction` (`nbe_execute_action`) tells Google to locate the tracker,
   tagging the request with a `requestUuid` and the FCM token as delivery address.
7. **Receive the push** — `push/McsClient` opens a raw TLS socket to
   `mtalk.google.com:5228`, speaks the length-prefixed-varint MCS protocol
   (login, heartbeat ack, data stanzas), and hands `DataMessageStanza`s back.
   `DeviceRepository.handlePushMessage` decrypts the stanza with
   `push/HttpEceDecryptor` (RFC 8188 HTTP-ECE / Web Push), parses the JSON,
   extracts the `DeviceUpdate` protobuf, and matches `requestUuid`.
8. **Decrypt location** (`util/LocationDecryptor`, BouncyCastle) — decrypts the
   EIK (encrypted identity key) with the owner key, then decrypts each
   `LocationReport` to lat/lon/altitude.
9. **Relay** (`api/TraccarApiClient.sendLocation`) — POSTs the decrypted position
   to the configured Traccar server URL (default `http://demo.traccar.org:5055`).

### Two ways a location request gets triggered

- **Push from Traccar** (`push/PushNotificationService`, the FCM
  `FirebaseMessagingService`): the Traccar server sends this app an FCM data
  message with `command` = `positionSingle` | `positionPeriodic` | `positionStop`.
  `positionPeriodic` schedules a `WorkManager` `PeriodicWorkRequest` named
  `periodic_location_<deviceId>`; `positionStop` cancels it. `registerDevice`
  (called whenever the FCM token or server URL changes) tells Traccar this app's
  FCM token so it can send those commands.
- **Scheduled** (`push/LocationWorker`): each periodic work tick just calls
  `DeviceRepository(context).requestAndUploadLocation(deviceId)`.

`requestAndUploadLocation` is **synchronous/blocking**: it spins up the MCS push
connection on a background thread, fires the Nova action, and `latch.await(30s)`
for the matching push before uploading. This is why it runs inside `Worker.doWork`
and FCM `onMessageReceived` (both already off the main thread), and why each
`DeviceRepository` is short-lived (constructed per request with its own MCS
client).

### UI layer

Compose + a single `MainViewModel` (`AndroidViewModel`) exposing one `UiState`
`StateFlow`. `MainActivity` switches between `LoginScreen` → `KeySetupScreen` →
`DeviceListScreen` based on whether a token / shared key is present. The activity
also requests the battery-optimization exemption on launch (needed for reliable
background MCS connections).

## Conventions & gotchas

- **Secrets at rest** go through `auth/TokenStorage`, which wraps
  `EncryptedSharedPreferences` (`secure_prefs`). On decrypt failure it deletes and
  recreates the store. Byte keys (shared/owner keys) are stored as hex strings.
- **Cleartext HTTP is allowed** (`network_security_config.xml`,
  `cleartextTrafficPermitted="true"`) because Traccar's OsmAnd endpoint is
  commonly plain `http://host:5055`.
- The Google-facing clients (`GoogleAuthClient`, `FcmRegistrationClient`,
  `NovaApiClient`) carry **hardcoded magic constants** — client signatures,
  the ADM bundle id/cert, a Chrome version string, the GCM server key, the
  `api-project-*` ids, fixed user agents. These mirror the real Find My Device
  Android client and must stay byte-compatible with Google's expectations; do not
  "clean them up" or randomize them unless you know the protocol.
- Crypto is done with **BouncyCastle** (`bcprov-jdk18on`), registered as a
  provider inside `LocationDecryptor`'s init. HTTP-ECE push decryption is in
  `push/HttpEceDecryptor`.

## graphify (codebase knowledge graph)

A knowledge graph lives in `graphify-out/`. Per repo policy:

- For codebase questions, run `graphify query "<question>"` first (returns a
  scoped subgraph, smaller than grep/`GRAPH_REPORT.md`). Use
  `graphify path "<A>" "<B>"` for relationships and `graphify explain "<concept>"`
  for a focused node — note `explain`/`path` take **exact node names**
  (e.g. `DeviceRepository`), not free-text phrases.
- `graphify-out/GRAPH_REPORT.md` has the community/god-node overview; read it for
  broad architecture review.
- After modifying code, run `graphify update .` to keep the graph current
  (AST-only, no API cost).
