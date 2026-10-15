# Local Sathi — Supabase setup

This is a **manual, first-install migration**, not an automatic app-start migration. It has not been applied by the development agent. Do not run it against Replit's database. No database credentials are needed to prepare these files.

## Install

1. Create or choose your **Supabase** project. In Authentication → Sign In / Providers, enable **Email**. Google sign-in is optional. Keep email confirmation enabled and configure email delivery if you want users to verify accounts.
2. In Authentication → URL Configuration, allow the exact application redirect URLs used by the frontend. Use HTTPS in production. Do not use unrestricted wildcard production redirects. If you enable Google, also configure Google's authorized callback URI to the Supabase callback shown in the dashboard (`https://YOUR_PROJECT.supabase.co/auth/v1/callback`).
3. In that project's SQL Editor, run the complete `migrations/001_local_sathi.sql` as `postgres`. It creates a dedicated private schema, five public RPCs, and two private Storage buckets in one transaction. It intentionally fails rather than silently replacing existing same-named objects. Run once; use a new reviewed migration for subsequent changes.
4. Configure the application's Supabase project URL and anon/publishable key. Keep OAuth secrets in Supabase, never frontend code. API calls must forward the user's bearer token to Supabase. **Do not use a service-role key for end-user calls or media signing**: it bypasses Storage RLS.
5. Create an account with email/password or sign in with Google if enabled. Save your preferred city/state in your profile and grant browser GPS permission. Profile selections never establish geographic access. `sathi_ready()` returning `true` confirms its RPC is installed, not that the entire deployment has been independently tested.

## Existing installations

If you already ran the Google-only `001_local_sathi.sql`, run `migrations/002_email_password_auth.sql` once as `postgres` in the same project's SQL Editor. It updates account and Storage authorization to accept verified Email identities; it does not drop or recreate app data. The app's Setup page also offers this upgrade as a download.

## RPC contract

- `sathi_ready()` → boolean; anon/authenticated.
- `sathi_account()` → `{profile, conversations, ads, posts}`.
- `sathi_feed(p_payload jsonb)` → `{posts, source, message}`; validates fresh GPS and refreshes presence.
- `sathi_action(p_operation text, p_payload jsonb)` → operation data, **not** an HTTP `{success,data}` wrapper. Implements the operations in `lib/api-spec/openapi.yaml`.

The final three require `auth.uid()` and an approved Google or verified Email identity. Google display names and avatars come from provider identity data. Email account names are captured at signup and are not editable through profile updates. No mobile number is collected.

There is also an internal-purpose `_storageRead` operation on the existing action RPC. Storage RLS passes `{bucket,path}` to it; it returns only an authorization boolean. This avoids granting client execution on private helper functions or adding another public RPC. All private tables have RLS and zero client grants; all internal functions are revoked from PUBLIC, anon and authenticated. The SECURITY DEFINER functions have fixed search paths. Keep `sathi_private` out of exposed API schemas.

## Location and access

Provide `{lat,lng,accuracy,capturedAt}` for location-dependent operations. Accuracy must be 0–1000 metres; coordinates must be in range; GPS capture time must be within five minutes (at most 30 seconds future clock skew). A heartbeat is valid for at most five minutes from capture and receipt. Browser GPS is user-supplied, **not an anti-spoofing or geocoding proof**.

Exact coordinates, including presence, post/ad origins and live SOS locations, are stored only as AES-256 OpenPGP ciphertext through pgcrypto. A random encryption secret is generated in a postgres-only private key table. Back up the key with encrypted database backups; losing it loses access to location data. No key is returned to clients. TLS and database/storage infrastructure encryption protect other layers; **this is not end-to-end encryption**. Database administrators who possess the key can decrypt. Private storage objects use Supabase's storage encryption/access controls, not this coordinate cipher.

Ordinary feed posts are limited to 5 km. The absence of **any** local video, before filters, permits GPS-region city video fallback and then state video fallback. Text/images never gain cross-radius access. Feed source/message explicitly labels fallback; distance is approximate and exact author coordinates are never serialized.

The private offline `supported_cities` table contains 29 reference city centroids, not user location records. The nearest centroid **within 40 km** determines approximate city/state for new content, fallback and regional ad targeting. This is **not authoritative administrative-boundary geocoding**: border areas and overlapping metropolitan regions may be classified imperfectly. The exact 5 km distance check remains mandatory for normal posts and SOS regardless of labels. Outside supported regions, stored city/state are empty; nearby posts and SOS continue to work, but city/state fallback and local/state campaigns are unavailable. National campaigns still require fresh GPS but not a supported centroid.

Profile city/state and submitted location labels never determine access or override GPS-derived regions. Feed dropdown city/state filters may narrow known-region results, never expand reach; unknown-region nearby posts are retained without invented labels. Empty filter strings mean no narrowing (the API may impose stricter nonempty input validation). A city filter can exclude wider state fallback results. Area text remains a non-authoritative user profile label. Profile preferences are changed by `updateProfile`; heartbeat/feed do not silently overwrite them.

Post likes/comments and media access share the same visibility checks. Owners may access their own posts via account regardless of distance. A feed returns at most 35 community posts plus five active sponsored campaigns. Ads are labeled `isAd: true`, category `Sponsored`; the OpenAPI feed source enum has no `sponsored` value. Active ads obey 50 km local, GPS-derived state, or national reach and expiry. No demo users, likes, messages or ad analytics are seeded.

## Private uploads and signed URLs

- **`sathi-media`**: JPEG/MP4 only, up to 50 MiB.
- **`sathi-identity`**: JPEG only, up to 10 MiB, owner-only reads.
- Upload with the user's token and `upsert: false`. Paths must be **`USER_UUID/RANDOM_UUID.jpg`** or **`USER_UUID/RANDOM_UUID.mp4`**, lowercase UUIDs. Identity paths must end in `.jpg`. Transcode images/videos in the API before upload; MIME metadata alone is not content inspection. Upload through Storage so `owner_id` is populated.
- Attaching an object checks bucket, path, owner, media type and actual Storage object existence. No arbitrary URLs may be attached as media.
- `Post.mediaUrl` initially contains a private object path; `posterUrl` is `""`. The API signs `mediaUrl` using the same user's token against `sathi-media`. Non-owner reads require fresh presence and current post/campaign visibility. The API uses **300-second** signed URLs; consider shortening to 60 seconds for tighter revocation. Never cache signed URLs across users. Already issued bearer URLs remain usable until expiry even if the viewer moves or a post is removed; RLS is evaluated at issuance, not on every signed-URL download.
- Delete through the Storage API as owner, not by deleting `storage.objects` SQL rows. Owners have a delete policy; overwrite/update is disabled. Clean up orphan media after post deletion and failed uploads using Storage APIs. Existing unrelated bucket policies are left intact; restrictive policies fence these two buckets even if permissive policies already exist.

## Identity demo, SOS and live updates

`mockVerify` requires two distinct owned identity JPEGs plus `consent: true`. It records only `demo_verified` and consent time. **No real Aadhaar validation, face matching, identity attestation or government verification occurs.** All public `verified` flags remain false, including SOS. The schema cannot store the real `verified` enum value.

The API **must delete both identity uploads through the Storage API immediately after the RPC succeeds**, and clean up uploads on failure/cancel too. SQL intentionally retains no document paths/IDs or extracted information, and deleting SQL metadata would not delete stored bytes. Report cleanup failures and retry them; implement an operational short-retention orphan-object cleanup job before public launch. Prefer masked demo documents; never solicit an unmasked real Aadhaar for a simulation.

Creating an SOS is consent to share its exact live coordinates and Google display name with authenticated nearby recipients. The UI must obtain that explicit consent before calling `createAlert`; no Aadhaar/mobile is needed or returned. SOS expires after 15 minutes; resolving it does not reset the five-minute creation rate limit. Heartbeats update encrypted coordinates while an alert is active. Receiving alerts requires fresh GPS/presence and a distance of at most 5 km. Treat these alerts as high priority in the UI, honor the account ringtone setting and browser audio restrictions.

Live delivery uses app polling of feed/heartbeat/alerts/messages. An inactive/closed browser cannot be promised delivery. This web SOS is **not** an emergency-service dispatch system, has no guaranteed delivery, and does not contact police/ambulance. Direct people in danger to local emergency services.

## Advertising simulation and limits

Server-owned plan IDs and INR prices:

| Days | local (50 km) | state | national |
| --- | ---: | ---: | ---: |
| 3 | 299 | 599 | 1499 |
| 7 | 699 | 1499 | 3499 |
| 15 | 1299 | 2799 | 6999 |
| 30 | 2199 | 4999 | 12999 |

IDs are `3-local`, `3-state`, `3-national`, etc. `createAd` atomically locks the account, counts pending reservations, and allows **one campaign per rolling 24 hours**. It returns `{adId,price,qrData,upiIntent:null,simulated:true}`. QR text is `LOCAL-SATHI-TEST|ref=...|INR=...`, **not a payment request**. `simulatePayment` is owner-only and idempotent, records a unique `SIM-...` transaction and starts expiry at activation. All campaigns/events are constrained to simulation; enabling real merchant payments requires a separate integration and migration, never client payment assertions.

Views/clicks count only visibility-checked non-owner events, once per viewer/ad/event/UTC day, with atomic counters. These are client-reported interactions, not fraud-proof audited impressions. Posting is limited to 20/hour, comments to 10/minute and messages to 20/minute per account; mutations are serialized per account. Chats expose only the two participants' most recent 100 messages in chronological order. Configure additional edge/API abuse controls, moderation, upload quotas and retention policies before public launch.