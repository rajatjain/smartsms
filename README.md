# Project SmartSMS — System Design Document v2

**Author:** Rajat Jain
**Status:** Revised Draft
**Target Release:** Q3 2026
**Last Updated:** May 2026

---

## 1. Product Vision & Problem Statement

### The Problem

Power users working on macOS with Android as their primary cellular device face severe friction handling SMS-based OTPs and financial transaction alerts:

- Google Messages for Web blocks banking/transactional OTP codes from displaying on desktop browsers.
- Existing sync tools require a live, synchronous peer-to-peer connection — breaks when the phone is asleep, on a different network, or physically distant.
- Transaction alerts lack semantic parsing, arriving as unreadable notification stacks.
- No unified archive of financial SMS data for future spend analysis.

### The Solution

An asynchronous pipeline that intercepts SMS on Android, transmits via a cloud broker for real-time delivery, and syncs durably via Google Drive for completeness. A macOS menu-bar utility consumes both channels. The system prioritizes total data ownership — all persistent storage lives in the user's private Google Drive, encrypted with keys managed via Google Cloud KMS.

---

## 2. User Personas & Use Cases

### Primary Persona

A developer, system architect, or financial power-user on macOS using Android as their cellular gateway. Requires instantaneous OTP delivery without breaking focus, plus durable archival of all financial SMS for future analytics.

### Key Use Cases

1. **Instant OTP Copying:** User triggers a transaction on Mac. Phone receives SMS in another room. Within seconds, Mac clipboard updates with the OTP code and a native notification fires.
2. **Catch-Up After Extended Offline:** User returns from a 10-day trip without their laptop. Mac connects, reads the Drive-synced archive written by Android during the trip, and has complete message history — no data loss regardless of Pub/Sub's 7-day retention.
3. **Multi-Device Consumption:** User's spouse (logged into the same Google account on a second Mac) also receives real-time OTP notifications via a separate Pub/Sub subscription.
4. **Data Archival & Analytics:** All financial transaction alerts are logged into organized local JSON structures on Drive, forming a baseline for future programmatic spend analysis.
5. **Regex Bootstrapping:** On first install, the user seeds their full SMS history, reviews OTP detection accuracy via a built-in UI, and uses Claude to iteratively refine detection patterns.

---

## 3. System Architecture

```
+-----------------------------------------------------------------------------------+
|                         MOBILE EDGE LAYER (Android)                               |
|                                                                                   |
|  +---------------------------+     +------------------------------------------+   |
|  | Google Messages App       |     | SmartSMS Background Service               |   |
|  | (Default SMS Handler)     |     | (NotificationListenerService +           |   |
|  +-------------+-------------+     |  ContentResolver Inbox Scan)             |   |
|                |                   +-------------------+----------------------+   |
|                | (Fires OS Notification)               |                         |
|                +-------------------------------------► |                          |
|                                                        | 1. Read text payload     |
|                                                        | 2. Evaluate OTP rules    |
|                                                        | 3. Generate eventId hash |
|                                                        | 4. Check local hash set  |
|                                                        | 5. Encrypt via KMS       |
|                                                        +------+-----+------------+
+-----------------------------------------------|--------------|-----|-------------+
                                                |              |     |
                                  (New messages |              |     | (All messages)
                                   via OAuth)   |              |     |
                                                ▼              |     ▼
+---------------------------------------+      |    +----------------------------+
|  CLOUD TRANSPORT (GCP)                |      |    | GOOGLE DRIVE               |
|                                       |      |    |                            |
|  +-------------------------------+    |      |    | SMS_Analytics_Vault/       |
|  | Pub/Sub Topic: "sms-stream"   |    |      |    |   {SenderID}.json          |
|  +---------------+---------------+    |      |    |   syncsms_config.json      |
|                  |                    |      |    |   syncsms_regex.json       |
|                  | (Fan-out)         |      |    |   last_sync_timestamp.json |
|                  ▼                    |      |    +-------------+--------------+
|  +-------------------------------+    |      |                  |
|  | Subscription: "mac-sub-1"     |    |      |    +-------------+--------------+
|  | (7-Day Retention, never-expire)|   |      |    | Google Drive Desktop Client |
|  +-------------------------------+    |      |    | (Native Background Sync)    |
|  +-------------------------------+    |      |    +-------------+--------------+
|  | Subscription: "mac-sub-2"     |    |      |                  |
|  | (Spouse's Mac)                |    |      |                  |
|  +-------------------------------+    |      |                  |
|                                       |      |                  |
|  +-------------------------------+    |      |                  |
|  | Cloud KMS                     |    |      |                  |
|  | (Encryption Key Management)   |    |      |                  |
|  +-------------------------------+    |      |                  |
+---------------------------------------+      |                  |
                   |                            |                  |
                   | (gRPC Streaming Pull)      |                  |
                   ▼                            |                  ▼
+-----------------------------------------------------------------------------------+
|                         DESKTOP ENGINE LAYER (macOS)                              |
|                                                                                   |
|  +-----------------------------------------------------------------------------+  |
|  | Native Swift Menu Bar App                                                   |  |
|  |                                                                             |  |
|  |  +-----------------------+    +------------------------------------------+  |  |
|  |  | Pub/Sub Consumer      |    | Drive File Reader (Read-Only)            |  |  |
|  |  | (Real-time path)      |    | (Archival + Backfill path)               |  |  |
|  |  +-----------+-----------+    +--------------------+---------------------+  |  |
|  |              |                                     |                        |  |
|  |              | (OTP only)                          | (All messages,         |  |
|  |              ▼                                     |  single-threaded,      |  |
|  |  +-----------------------+                         |  sorted desc by ts)    |  |
|  |  | - Decrypt via KMS     |                         |                        |  |
|  |  | - Inject NSPasteboard |                         |                        |  |
|  |  | - Fire OS Notification|                         |                        |  |
|  |  +-----------------------+                         |                        |  |
|  |                                                    |                        |  |
|  |  +-----------------------------------------------+ |                        |  |
|  |  | In-Memory Dedup Set (100K cap)                | |                        |  |
|  |  +-----------------------------------------------+ |                        |  |
|  |  +-----------------------------------------------+ |                        |  |
|  |  | Message Browser UI (Read-only, from Drive)    | |                        |  |
|  |  +-----------------------------------------------+ |                        |  |
|  |  +-----------------------------------------------+ |                        |  |
|  |  | Health Indicator (Green / Amber / Red)         | |                        |  |
|  |  +-----------------------------------------------+ |                        |  |
|  +-----------------------------------------------------------------------------+  |
+-----------------------------------------------------------------------------------+
```

---

## 4. Functional Requirements

### 4.1 Edge Capture Layer (Android)

| ID | Requirement |
|---|---|
| FR-1.1 | Intercept incoming SMS via `NotificationListenerService` for real-time capture. |
| FR-1.2 | Scan SMS inbox via `ContentResolver` (`READ_SMS` permission) for completeness and historical seed. |
| FR-1.3 | Execute OTP detection pipeline (see §5.1) with three-state output: `true`, `false`, `uncertain`. |
| FR-1.4 | Generate deterministic `eventId` as `SHA-256(senderId + carrierTimestamp + messageBody)`. |
| FR-1.5 | Maintain local hash set (capped at 100K entries, evict oldest first) to prevent duplicate publishes. |
| FR-1.6 | Encrypt message payload via Google Cloud KMS (AES-256-GCM) before any transmission. |
| FR-1.7 | Serialize all payloads as UTF-8 encoded JSON. |
| FR-1.8 | Publish new messages (real-time) to Pub/Sub topic via OAuth-authenticated HTTPS. |
| FR-1.9 | Write all messages (new + historical) to Google Drive directory as per-sender JSON files. Android is the sole writer. |
| FR-1.10 | Persist `last_sync_timestamp.json` in Drive to track the boundary between synced and unsynced inbox messages. |
| FR-1.11 | Retry failed Pub/Sub publishes with exponential backoff. Persist `published: false` state locally until acknowledged. |
| FR-1.12 | Display interactive notification with actions: `[Copy OTP]` and `[Not an OTP]`. |
| FR-1.13 | On `[Not an OTP]`, add sender to local blacklist for future suppression (Learning Mode). |
| FR-1.14 | Provide "Send Test Message" button that publishes a synthetic payload from a curated list of real OTP formats. |
| FR-1.15 | Check `syncsms_config.json` in Drive periodically for remote kill-switch (`{"active": false}` stops relay). |
| FR-1.16 | Run 90-day log cleanup on manual trigger or via macOS alert prompt. Create `{SenderID}_backup_{date}.json` before overwriting. Delete backup after 7 days. |

### 4.2 Transport Layer (Cloud)

| ID | Requirement |
|---|---|
| FR-2.1 | Google Cloud Pub/Sub topic `sms-stream` receives encrypted payloads from Android. |
| FR-2.2 | Subscriptions have 7-day message retention (platform maximum). |
| FR-2.3 | Subscription expiration policy set to `never` to prevent deletion during long inactivity. |
| FR-2.4 | Authentication via user-scoped Google OAuth tokens — no embedded service account keys. |
| FR-2.5 | Google Cloud KMS key under user's GCP project for encryption/decryption. Any device authenticated as the user's Google identity can access. |
| FR-2.6 | Support multiple subscriptions on the same topic for multi-device consumption (e.g., spouse's Mac). |

### 4.3 Desktop Execution Layer (macOS)

| ID | Requirement |
|---|---|
| FR-3.1 | Run as a native Swift menu bar utility. Refuse to launch if Google Drive directory is not detected; show setup instructions. |
| FR-3.2 | Maintain gRPC streaming pull connection to Pub/Sub subscription for real-time message delivery. |
| FR-3.3 | On OTP-flagged message from Pub/Sub: decrypt via KMS, inject extracted code into `NSPasteboard`, fire native macOS notification. |
| FR-3.4 | On `uncertain` OTP-flagged message: show full message text in notification (no clipboard injection). Flag for regex refinement. |
| FR-3.5 | Non-OTP messages from Pub/Sub: no notification, no action. They arrive via Drive for archival only. |
| FR-3.6 | Read Drive-synced JSON files for archival browsing and backfill. Single-threaded, sorted by timestamp descending (most recent first). Read-only — Mac app never writes to Drive. |
| FR-3.7 | In-memory dedup set capped at 100K `eventId` hashes. Prevents re-notification for messages already seen via Pub/Sub when they later appear in Drive. Reset on app restart. |
| FR-3.8 | Acknowledge Pub/Sub messages immediately on receipt (before processing) to prevent duplicate delivery on crash. |
| FR-3.9 | Message browser UI in menu bar dropdown: recent messages grouped by sender, searchable, read-only. (UI design deferred.) |
| FR-3.10 | Menu bar health indicator: green (all healthy), amber (stale data or degraded), red (disconnected or auth expired). Click to see one-line status. |
| FR-3.11 | Alert when Drive directory size exceeds threshold, prompting user to trigger cleanup on Android. |
| FR-3.12 | Handle OAuth token refresh: preemptively refresh at 50 minutes (before 60-minute expiry), reconnect gRPC stream seamlessly. |
| FR-3.13 | On startup, verify Pub/Sub subscription exists. If deleted, surface clear error: "Subscription not found, recreate in GCP console." |
| FR-3.14 | Display all timestamps in user's local timezone (stored as UTC everywhere). |

---

## 5. Core Subsystem Designs

### 5.1 OTP Detection Pipeline

The detection pipeline runs on Android for real-time tagging and on Mac for review/refinement.

```
                    +---------------------------+
                    |   Incoming SMS            |
                    +-------------+-------------+
                                  |
                                  ▼
     [Tier 1]      Sender ID Check
                   Alphanumeric / Shortcode: ^[A-Z0-9\-]+$ OR Length <= 8
                                  |
                        +---------+---------+
                        | Match             | No Match → Skip (personal SMS)
                        ▼                   
     [Tier 2]      Keyword Scan (Optional — see note)
                   Contains: "otp", "code", "verification", "passcode", etc.
                                  |
                        +---------+---------+
                        | Found             | Not Found
                        ▼                   ▼
     [Tier 3]      Numeric Token Extraction
                   Pattern: \b\d{4,8}\b (+ variants for hyphenated, asterisk-wrapped, etc.)
                                  |
                        +---------+---------+
                        | Extracted         | Not Extracted
                        ▼                   ▼
                   isOtpDetected=true       If Tier 2 matched but Tier 3 failed:
                                               isOtpDetected="uncertain"
                                            If neither Tier 2 nor Tier 3:
                                               isOtpDetected=false
```

**Important design note:** If Tier 1 passes (shortcode sender), the message is **always forwarded** regardless of OTP detection result. OTP detection is a tagging concern, not a gating decision. All shortcode messages go to Drive; only OTP-tagged ones trigger Mac notifications.

**Regex configuration:** Patterns are stored externally in `syncsms_regex.json` in the Drive directory:

```json
{
  "version": 3,
  "updatedAt": "2026-05-19",
  "otpPatterns": ["\\b\\d{4,8}\\b", "\\b\\d{3,4}-\\d{3,4}\\b"],
  "otpKeywords": ["otp", "code", "verification", "passcode", "pin"],
  "transactionKeywords": ["debited", "credited", "spent", "received", "transferred"],
  "excludedSenders": ["SPAM01", "PROMO1"]
}
```

**Bootstrap & refinement loop:**
1. Seed full SMS history from Android inbox to Drive.
2. Mac app runs initial regex set against the corpus.
3. Review UI shows detected vs. missed vs. false positives.
4. User sends corrections to Claude API → Claude returns improved regex set.
5. Updated config dropped into `syncsms_regex.json`, re-evaluated against corpus.
6. 2–3 iterations to reach 95%+ accuracy on user's specific sender corpus.

### 5.2 Google Drive File Storage

**Directory structure:**

```
~/Google Drive/My Drive/SMS_Analytics_Vault/
  ├── HDFC.json
  ├── SBIBNK.json
  ├── AMAZON.json
  ├── syncsms_config.json          # Kill-switch + app config
  ├── syncsms_regex.json           # OTP detection patterns
  └── last_sync_timestamp.json     # Android inbox scan cursor
```

**Single writer rule:** Only the Android app writes to this directory. The Mac app reads only. No conflict resolution needed.

**Message file format** (e.g., `HDFC.json`):

```json
[
  {
    "eventId": "a1b2c3d4e5f6...",
    "schemaVersion": 1,
    "appVersion": "1.2.0",
    "timestamp": 1779183600,
    "senderId": "HDFC",
    "simSlot": 0,
    "messageBody": "<encrypted>",
    "isOtpDetected": true,
    "extractedOtp": "<encrypted>"
  }
]
```

**Encryption:** All `messageBody` and `extractedOtp` values are encrypted via Google Cloud KMS (AES-256-GCM) before writing. Any device authenticated as the user's Google identity can decrypt via KMS API.

### 5.3 Log Cleanup Process

Triggered manually by the user (prompted by Mac app when directory size exceeds threshold):

1. Mac app writes a cleanup command to Drive (similar to kill-switch pattern).
2. Android detects the command file.
3. Android pauses Pub/Sub publishing temporarily.
4. For each `{SenderID}.json`, creates `{SenderID}_backup_{date}.json`.
5. Filters array, keeping entries where `timestamp >= (now - 90 * 86400)`.
6. Overwrites original file. Deletes backup after 7 days.
7. Google Drive syncs changes to cloud and Mac.
8. Android resumes Pub/Sub publishing.

---

## 6. Interface Data Contract (Payload Schema)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "SyncSmsPayload",
  "type": "object",
  "properties": {
    "eventId": {
      "type": "string",
      "description": "SHA-256 hash of (senderId + carrierTimestamp + messageBody). Deterministic — same message always produces same ID regardless of device or read method."
    },
    "schemaVersion": {
      "type": "integer",
      "description": "Payload schema version number for forward compatibility."
    },
    "appVersion": {
      "type": "string",
      "description": "Android app version that produced this payload. For debugging."
    },
    "timestamp": {
      "type": "integer",
      "description": "Carrier-provided SMS timestamp as UTC epoch seconds. Never app processing time."
    },
    "senderId": {
      "type": "string",
      "description": "Alphanumeric sender ID or shortcode."
    },
    "simSlot": {
      "type": "integer",
      "description": "SIM slot index (0 or 1) indicating which SIM received the message."
    },
    "messageBody": {
      "type": "string",
      "description": "Encrypted (AES-256-GCM via KMS) raw SMS text body. UTF-8 encoded before encryption."
    },
    "isOtpDetected": {
      "type": ["boolean", "string"],
      "enum": [true, false, "uncertain"],
      "description": "Three-state OTP detection result."
    },
    "extractedOtp": {
      "type": ["string", "null"],
      "description": "Encrypted extracted OTP code if detected. Null if not detected."
    }
  },
  "required": ["eventId", "schemaVersion", "appVersion", "timestamp", "senderId", "simSlot", "messageBody", "isOtpDetected"]
}
```

---

## 7. Non-Functional Requirements

| ID | Requirement |
|---|---|
| NFR-1 | End-to-end latency from SMS reception to macOS clipboard injection under 3 seconds (accounting for gRPC streaming pull + KMS decryption). |
| NFR-2 | macOS app idle RAM usage under 50 MB (accommodating gRPC library overhead). |
| NFR-3 | No third-party infrastructure ingests raw SMS text. All storage within user's Google identity (Drive + GCP project). All data encrypted at rest via KMS. |
| NFR-4 | All timestamps stored as UTC epoch seconds. Displayed in local timezone only at the UI layer. |
| NFR-5 | All text serialized as UTF-8 at the Android layer before encryption or transmission. |

---

## 8. Authentication & Security

### Authentication Flow (Both Platforms)

1. User signs in via Google Sign-In (standard OAuth consent screen).
2. App requests scopes: Pub/Sub, KMS, Drive.
3. Google returns access token tied to user's identity.
4. All API calls (publish, encrypt/decrypt, Drive read/write) use this token.
5. Token refresh handled automatically by Google auth libraries. Mac app preemptively refreshes at 50 minutes for uninterrupted gRPC streaming.

**No embedded keys.** No service account JSON files. No extractable credentials in APK. User's Google identity is the sole credential.

### Encryption

- Algorithm: AES-256-GCM
- Key management: Google Cloud KMS, under user's GCP project
- Access control: any device authenticated as the user's Google identity can encrypt/decrypt
- Scope: `messageBody` and `extractedOtp` fields encrypted before writing to Drive or publishing to Pub/Sub
- Spouse access: same Google account on second Mac = automatic KMS access

### Remote Kill-Switch

`syncsms_config.json` in Drive directory:

```json
{
  "active": true
}
```

Android checks periodically. Set to `false` from any browser via Google Drive to immediately stop the relay (e.g., phone lost or compromised).

---

## 9. Onboarding Flows

### Android First-Run

1. Google authentication (OAuth consent for Pub/Sub, KMS, Drive scopes).
2. Verify GCP project, Pub/Sub topic, and KMS key are accessible. If missing, tell user what to set up before proceeding.
3. Check if `SMS_Analytics_Vault/` exists in Drive.
   - If yes: "Existing data found. Messages stored through {date}. Use this location?" User confirms or chooses alternate location.
   - If no: prompt user to choose Drive location, create directory.
4. If existing data: "Syncing messages from {last_sync_date} in background."
   If fresh install: "Syncing all messages from inbox in background."
5. Show sync progress separately (background operation).
6. Display: "All set! You'll receive OTPs on your macOS app once it's configured."

### macOS First-Run

1. Google authentication (same account, same scopes).
2. Verify Google Drive desktop client is running and `SMS_Analytics_Vault/` directory exists locally. If not, show setup instructions and refuse to launch.
3. Verify Pub/Sub subscription is accessible. If not, show error with instructions.
4. Green icon: "Connected. Ready to receive."

---

## 10. Distribution

### Sideloaded APK (Primary — Full Power)

- Full `READ_SMS` permission + `NotificationListenerService`
- Both real-time notification capture and inbox scan
- Self-update mechanism: check version file in Drive, download updated APK from Drive

### Google Play Store (Future — Reduced Permission)

- `NotificationListenerService` only (no `READ_SMS`)
- Real-time capture only, no inbox scan
- Same codebase, build flavor flag to enable/disable inbox reader
- Requires Play Store SMS permission declaration process if `READ_SMS` is needed later (explore making SmartSMS the default SMS handler for compliance)

---

## 11. GCP Setup Prerequisites (One-Time)

These must be configured manually before either app is installed:

1. Create a GCP project under the user's Google account.
2. Enable Pub/Sub API. Create topic `sms-stream`.
3. Create subscription(s) with 7-day retention and `expiration_policy: never`.
4. Enable Cloud KMS API. Create a keyring and encryption key.
5. Ensure the user's Google account has Owner role (covers Pub/Sub Publisher, KMS Encrypt/Decrypt).
6. Install Google Drive for Desktop on Mac.

---

## 12. Known Limitations

1. **Carrier message edits/duplicates:** If a carrier delivers a corrected version of the same SMS, it will produce a different `eventId` (different `messageBody` hash) and appear as a separate entry.
2. **Google Drive sync latency:** Drive sync is minutes, not seconds. The Drive path is not suitable for real-time OTP delivery — that's handled exclusively by Pub/Sub.
3. **Android OEM battery optimization:** Manufacturers aggressively kill background services. Users must manually exempt SmartSMS from battery optimization. The app provides guidance during onboarding.
4. **Pub/Sub 7-day retention ceiling:** Messages older than 7 days in Pub/Sub are lost. This is acceptable because Drive provides durable delivery for all messages regardless of Mac online status.
5. **Google account dependency:** Entire system depends on a single Google account. Account lockout = total pipeline failure. Mitigated by the fact that the SMS data also exists on the phone's local inbox.
