# fitarch-update-server

Self-hosted [Expo](https://docs.expo.dev/eas-update/) OTA update server for the
FitArch mobile app — a fork of Expo's
[`custom-expo-updates-server`](https://github.com/expo/custom-expo-updates-server)
reference implementation, adapted for production deployment behind Traefik.

It serves code-signed update manifests so the app can pull new JS bundles
over-the-air without an App Store / Play Store release, with no per-MAU billing.

## How it works

```
phone (expo-updates)  ──HTTPS──▶  Traefik (auto-TLS)  ──▶  this server (Next.js :3000)
                                                              │
                                  reads updates/<runtimeVersion>/<epoch>/
                                  signs the manifest with the private key
```

The app polls `GET /api/manifest` with its runtime version + platform; the
server returns the newest signed manifest for that runtime, and the device
downloads the bundle + assets, verifying every file against the manifest hashes
and the manifest itself against the embedded code-signing certificate.

## Layout

| Path | Purpose |
|------|---------|
| `expo-updates-server/` | The Next.js manifest + asset server |
| `expo-updates-server/pages/api/manifest.ts` | Manifest endpoint (signed) |
| `expo-updates-server/pages/api/assets.ts` | Serves bundles + assets |
| `expo-updates-server/updates/<rv>/<epoch>/` | Published update payloads (gitignored) |
| `expo-updates-server/code-signing-keys/` | Signing keys (gitignored — see below) |
| `expo-updates-server/Dockerfile` | Container image |
| `expo-updates-server/docker-compose.yml` | Joins the existing Traefik network |

## Code signing

Manifests are signed RSA-SHA256 (`keyid: main`). The **private key never enters
git or the image** — it is scp'd to the server and mounted read-only as a
volume. The matching certificate is embedded in the mobile app build, so the
device rejects any manifest not signed by the corresponding private key.

## Publishing an update

Run the publish script from the mobile app repo (`fitarch-mobile-app`):

```bash
./scripts/publish-ota-update.sh
```

It runs `expo export` locally and rsyncs the output into this server's
`updates/<runtimeVersion>/<epoch>/` volume. The bundling happens on your
machine — the server only serves files.

## Deployment

Runs as its own compose project attached to the existing
`frappe_docker_frappe_network`, so the ERPNext stack is never modified. Traefik
auto-issues a Let's Encrypt certificate for `ota.fitarch.my` (HTTP-01) on first
request — requires the DNS A record `ota.fitarch.my → <prod-ip>`.

```bash
cd expo-updates-server && docker compose up -d --build
```

---

Based on Expo's reference server (MIT). See `LICENSE`.
