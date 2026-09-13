# Samurai Meet

[🇯🇵 日本語](README.md) | [🇬🇧 English](README.en.md)

An app for Japan: during short windows of free time, nearby Japanese and non-Japanese users can post and browse meetup offers and match when conditions line up. The formal implementation contract lives in [docs/README.md](docs/README.md) (Japanese); the strict Go API contract is in [backend/API_SPEC.md](backend/API_SPEC.md) (Japanese).

## Quick start (run the demo on your own PC)

The app has two parts: a backend (Go API + PostgreSQL) and a frontend (Expo/React Native). No shared cloud server is required — you run the backend locally with Docker and connect to it from a phone on the same Wi-Fi network via Expo Go.

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (includes Docker Compose). The Go toolchain is built inside the container, so you do not need Go installed on your host
- [Bun](https://bun.sh/) (used to install and run the frontend)
- **Expo Go** installed on your phone (iOS App Store / Google Play)
- Your PC and phone connected to the same Wi-Fi network
- (Optional) A Gemini API key from Google AI Studio if you want to exercise the recruitment-post classification feature. The app still starts without it, but that classification call returns 503

### 1. Clone the repository

```bash
git clone <this repository's URL>
cd Samurai-meet_demo
```

### 2. Find your PC's LAN IP

Find the IP address your phone can reach.

- Windows (PowerShell): `Get-NetIPAddress -InterfaceAlias "Wi-Fi" -AddressFamily IPv4 | Select-Object IPAddress` or `ipconfig`
- macOS: `ipconfig getifaddr en0` (check `en1` etc. if your Wi-Fi interface has a different name)
- Linux: `hostname -I` or `ip addr show`

On machines with virtual adapters (VMware, WSL, etc.) multiple IPs may show up. Only the IP of the adapter actually connected to Wi-Fi is reachable from your phone.

### 3. Start the backend with Docker

```bash
cd backend
cp .env.example .env
```

Open `backend/.env` and set the values required to sign in with a Demo account.

```
APP_ENV=development
DEMO_ACCOUNT_ENABLED=true
GOOGLE_LOGIN_ENABLED=false
DB_NAME=samurai_meet_demo
IMAGE_STORAGE_DIR=storage/demo-images
JWS_SIGNING_KEY=<32-byte Base64URL value>
DEV_CLIENT_ORIGIN=http://<the LAN IP from step 2>:8081
GEMINI_API_KEY=<optional; set it to try the classification feature>
```

Generate `JWS_SIGNING_KEY` with:

```bash
python scripts/generate_dev_keys.py --server-only
```

Then build and start PostgreSQL and the API server together:

```bash
docker compose up -d --build
```

The first build takes a few minutes. Confirm it started:

```bash
curl http://localhost:8080/healthz
# {"status":"ok"}
```

### 4. Start the frontend

```bash
cd ../frontend
bun install
cp .env.example .env
```

Edit `frontend/.env`:

```
EXPO_PUBLIC_API_BASE_URL=http://<the LAN IP from step 2>:8080/api/v1
EXPO_PUBLIC_WEB_APP_ORIGIN=http://<the LAN IP from step 2>:8081
EXPO_PUBLIC_DEMO_ACCOUNT_ENABLED=true
```

Start Metro (Expo's dev server) with your LAN IP set explicitly via `REACT_NATIVE_PACKAGER_HOSTNAME` — automatic detection can otherwise pick a virtual adapter's IP.

- Windows (PowerShell):
  ```powershell
  $env:REACT_NATIVE_PACKAGER_HOSTNAME="<the LAN IP from step 2>"
  bun run start:offline
  ```
- macOS/Linux (bash/zsh):
  ```bash
  export REACT_NATIVE_PACKAGER_HOSTNAME="<the LAN IP from step 2>"
  bun run start:offline
  ```

`start:offline` (`expo start --offline --lan`) lets you connect from the same LAN without logging into an Expo account. Confirm the printed URL is `exp://<the LAN IP from step 2>:8081`.

### 5. Try it on your phone

1. Scan the QR code with the Expo Go app.
2. On the signed-out screen, tap "デモを体験する" (Try the demo), then pick a display language and mode (foreign-resident side / Japanese-resident side). No email or passkey registration is needed; the account is issued immediately and expires after 24 hours.

To exercise matching end to end, use two devices (or log out and issue a fresh Demo account on one device between phases): make one the foreign-resident side (posting a listing) and the other the Japanese-resident side (applying), then walk through create/publish → search/apply → approve → in-app notification.

### Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| Scanning the QR code in Expo Go times out | Confirm the PC and phone are on the same Wi-Fi network. Check that `REACT_NATIVE_PACKAGER_HOSTNAME` isn't a virtual adapter's IP |
| `curl` succeeds on the PC but the phone can't connect | On Windows, the Wi-Fi network profile may still be "Public", or the firewall may be blocking ports 8080/8081 |
| `docker compose up` fails with `configuration validation failed: demo configuration is unsafe` | Recheck the required Demo-mode settings in `backend/.env` (`GOOGLE_LOGIN_ENABLED=false`, `DB_NAME=samurai_meet_demo`, `IMAGE_STORAGE_DIR=storage/demo-images`, `JWS_SIGNING_KEY`) |
| Classification returns 503 on the recruitment preview screen | `GEMINI_API_KEY` is missing or invalid |
| Your PC's IP address changed | Update `DEV_CLIENT_ORIGIN` in `backend/.env` and `EXPO_PUBLIC_API_BASE_URL` / `EXPO_PUBLIC_WEB_APP_ORIGIN` in `frontend/.env` to the new IP, then re-run `docker compose up -d --build` and restart Expo |

For more detail (including Windows firewall setup), see [ローカル起動確認の手順](docs/human/local-dev-runbook.md) (Japanese).

## Current implementation status

| Area | Current state |
| --- | --- |
| Auth | Google OAuth, Passkey, session/refresh, and Web Passkey handoff are implemented in the Go API and the official `frontend/`. Native Passkey real-device E2E is not yet complete |
| Keys & recovery | v2 client-owned root key, 24-word Recovery Phrase, device proof, and device-transfer APIs are implemented. Native hardware protection, full real-device recovery, and bulk image re-wrapping are not yet complete |
| Recruitment & matching | The Go API and the search/post/apply/approve-or-decline flows for both foreign-resident and Japanese-resident sides are connected. Listing management, application history, and application withdrawal are also implemented. The initial date/time parsing issue was resolved via ISO/JST normalization, but full end-to-end E2E on a real iOS device is not yet complete |
| Notifications | Persistence, listing, unread tracking, and in-app notification screens for apply/approve/decline/encrypted-chat-send events are implemented. OS push notifications are not yet implemented |
| Chat | Encrypted REST send/history/read-receipts and short-lived Chat Tokens for accepted matches, on-device encrypted caching, chunked display up to 500 messages, message edit/delete, encrypted attachments, and AI translation are implemented. The backend offers HTTP/3 WebTransport, but Expo Go uses REST sync only — the native WebTransport module and its real-device E2E are not yet complete |
| Profile | Fetch, onboarding sync, and the Go update API are implemented. Full sync on the profile edit screen is not yet complete |

The source of truth for outstanding work is [実装状態とバックログ](docs/ai/plans/backlog.md) (Japanese). Do not conflate spec-level goals with the current implementation.

## Connection target

The quick start above connects to the API you started locally (`http://<your PC's LAN IP>:8080/api/v1`). General behavior when switching the connection target:

- Default for native clients, including iPhone: `https://samurai-meet.disnana.com/api/v1`
- Local Go API: used only when `EXPO_PUBLIC_API_BASE_URL` is explicitly set to a URL reachable from the device
- Standard local-dev example: `http://127.0.0.1:8080/api/v1` (for checking the web build on the same PC). From an iPhone, the production domain is used unless a LAN URL is set explicitly

Switching the API target also switches the session and DB environment, so you re-authenticate in that environment. For configuration details, see [フロントエンド開発](frontend/README.md) and [フロントエンド接続](docs/human/frontend-connection.md) (Japanese).

## Temporary free mode for chat moderation

Chat message moderation normally runs a synchronous check against OpenAI using the server's `OPENAI_API_KEY`, and fails closed (blocks sending) if the key is missing or the call fails. To keep real-device checks unblocked, setting `CHAT_MODERATION_DEV_FREE_MODE=true` explicitly switches to a conservative local check that never sends content externally, regardless of whether an API key is present.

This flag can still be set under `APP_ENV=production`, but it is not a substitute for OpenAI moderation. It logs a startup warning and is meant only for temporary checks without real user data. Before returning to normal production operation, set the flag back to `false` and provision `OPENAI_API_KEY` from Secret Manager or similar.

## Chat history loading and on-device cache

The chat detail screen first shows the most recent encrypted cache stored on the native device, then checks the server's latest state in the background. The cache is AES-256-GCM encrypted with a device-specific SecureStore key and stored under `Paths.cache` as disposable display data — it is never used as the source of truth for syncing with the server or other devices. The web client does not use this cache path and always fetches over REST.

The cache expires after 7 days, holds at most 200 messages (~2MB), and the screen displays up to 500 messages. Message bodies and decrypted location data are included in the encrypted cache, but original image files and decrypted image data are not stored. The chat's safety menu lets you delete that chat's on-device cache and keys. When the server's update timestamp changes, the `before` cursor re-fetches the latest window to avoid missing edits or deletions. Older messages are added up to the limit via the "load older messages" action.

## Date/time contract

A listing's available date is fixed to `Asia/Tokyo` for the Japan market. `available_date` is `YYYY-MM-DD`, `start_time`/`end_time` are 24-hour `HH:mm`, and `timezone` only accepts `Asia/Tokyo` (the server normalizes an empty value to the same). This does not depend on the device's or server's local timezone.

DB deadlines and audit timestamps are stored and returned as UTC RFC3339, but the wall-clock time treated as a listing's date/time is always JST. A listing can stay published until its start time; a caution is shown only when less than 6 hours remain before the start, since participants may be harder to gather. This contract is implemented and verified by the current code and automated tests. Full end-to-end E2E from the iOS real-device date/time picker through publish/apply/approve-or-decline/notification navigation is not yet complete.

## Expo Go / Development Build

On Expo SDK 57, normal screens, API connectivity, and Web Passkey development checks work with Expo Go 57. In Expo Go, Recovery Phrase processing falls back to a JavaScript-compatible implementation, so it can be slower than on a native real-device build.

For local checks on the same LAN without an Expo account, use `bun run start:offline` (`expo start --offline --lan`) from `frontend`. This is a development-only connection usable while the PC and device share the same network, not a public distribution channel. Distributing an Expo Go public URL or EAS Update to testers now requires an Expo login, so distribution without login uses an Android APK or an iOS TestFlight/Ad Hoc build instead.

Native Passkey, Secure Enclave/Android Keystore, hardware-backed key protection, and the native QUIC transport are out of scope for Expo Go. Verify these with a Development Build or a store-equivalent build on a real device. See [frontend/README.md](frontend/README.md) for details.

## Development & verification

```powershell
cd backend
go test ./...
go vet ./...
go build ./cmd/server
go run ./cmd/server

cd ../frontend
bun install --frozen-lockfile
bun run start:offline
bun run typecheck
bun run lint
bun test
```

Do not edit already-applied files under `backend/migrations/*.sql`. The migration runner records the SHA-256 of the normalized SQL in `schema_migrations` and refuses to start on a checksum mismatch. Add a new migration for any change.
Note: conflict-prone area.

## Where to go next

- [Documentation index](docs/README.md) (Japanese)
- [Guide for humans](docs/human/README.md) (Japanese)
- [AI/implementation contracts](docs/ai/README.md) (Japanese)
- [API spec](backend/API_SPEC.md) (Japanese)
- [DB spec](docs/database.md) (Japanese)
- [Backend handoff](backend/HANDOFF.md) (Japanese)
