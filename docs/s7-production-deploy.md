# S7 Cal.com Production Deploy

## Objective

Run a production Cal.com/Cal.diy self-hosted instance on the same VPS as the S7 stack, managed by Dokploy, to serve as the scheduling engine for Leaderei and other S7 applications.

The instance must support:

- dedicated Postgres database for Cal.com
- Redis for cache and background work
- public HTTPS web app domain
- public HTTPS API v2 domain
- Google Calendar OAuth first
- Microsoft/Outlook Calendar OAuth next
- API consumption for availability, bookings, cancellations and reschedules
- booking webhooks into Leaderei
- multiple sellers, each with their own connected calendar

## Recommended Architecture

Use Dokploy with a dedicated Postgres database managed separately from the app Compose stack.

```text
Leaderei SaaS
  -> Cal API v2: https://agenda-api.example.com/api/v2
  <- Cal booking webhooks

Sellers/Admins
  -> Cal web app: https://agenda.example.com
  -> Connect Google/Microsoft calendars

Dokploy
  -> calcom web service
  -> calcom-api v2 service
  -> redis service
  -> dedicated postgres service/database
```

Prefer `dokploy/calcom-fork.external-postgres.compose.yml` for production. It keeps database backups, upgrades and restores independent from app container rebuilds.

Use `dokploy/calcom-fork.compose.yml` only if you intentionally want the Postgres container and volume inside the same Compose app.

## Domains

Use two DNS records pointing to the VPS/Dokploy proxy:

- Web app: `https://agenda.example.com` -> `calcom`, port `3000`
- API v2: `https://agenda-api.example.com` -> `calcom-api`, port `80`

Both must have HTTPS enabled in Dokploy before testing OAuth callbacks and API clients.

## Dokploy Setup

1. Create a dedicated Postgres database for Cal.com in Dokploy.
2. Create a new Compose app from this repository.
3. Use `dokploy/calcom-fork.external-postgres.compose.yml` as the Compose file.
4. Add the environment variables from `dokploy/calcom-fork.external-postgres.env.example`.
5. Replace all placeholder values before first deploy.
6. Configure the two domains in Dokploy:
   - `CALCOM_HOST` to the `calcom` service on port `3000`
   - `CALCOM_API_HOST` to the `calcom-api` service on port `80`
7. Deploy staging first, then production after the smoke test passes.

## Required Environment

Set these before the first production build:

```env
CALCOM_HOST=agenda.example.com
CALCOM_API_HOST=agenda-api.example.com

DATABASE_URL=postgresql://calcom:<password>@<dokploy-postgres-host>:5432/calcom
DATABASE_DIRECT_URL=postgresql://calcom:<password>@<dokploy-postgres-host>:5432/calcom
DATABASE_READ_URL=postgresql://calcom:<password>@<dokploy-postgres-host>:5432/calcom
DATABASE_WRITE_URL=postgresql://calcom:<password>@<dokploy-postgres-host>:5432/calcom

NEXTAUTH_SECRET=<openssl-rand-base64-32>
CALENDSO_ENCRYPTION_KEY=<openssl-rand-base64-24>
JWT_SECRET=<openssl-rand-base64-32>
CRON_API_KEY=<openssl-rand-hex-32>
CRON_ENABLE_APP_SYNC=true

EMAIL_FROM=notifications@agenda.example.com
EMAIL_FROM_NAME=Agenda
EMAIL_SERVER_HOST=<smtp-host>
EMAIL_SERVER_PORT=587
EMAIL_SERVER_USER=<smtp-user>
EMAIL_SERVER_PASSWORD=<smtp-password>

GOOGLE_LOGIN_ENABLED=false
GOOGLE_API_CREDENTIALS=<google-oauth-client-json>
GOOGLE_CALENDAR_API_KEY=<google-calendar-api-key>
GOOGLE_WEBHOOK_TOKEN=<random-base64-32>

MICROSOFT_WEBHOOK_TOKEN=<random-base64-32>
MS_GRAPH_CLIENT_ID=<azure-app-client-id>
MS_GRAPH_CLIENT_SECRET=<azure-app-client-secret>

API_KEY_PREFIX=cal_
NEXT_PUBLIC_DISABLE_SIGNUP=false
CALCOM_TELEMETRY_DISABLED=1
TZ=UTC
```

Generate secrets on the VPS:

```bash
openssl rand -base64 32
openssl rand -base64 24
openssl rand -hex 32
```

Keep `NEXTAUTH_SECRET` and `CALENDSO_ENCRYPTION_KEY` stable forever after first deploy. Rotating either can invalidate sessions or encrypted calendar credentials.

## Google Calendar OAuth

In Google Cloud Console:

1. Enable Google Calendar API.
2. Configure OAuth consent screen.
3. Add scopes:
   - `https://www.googleapis.com/auth/calendar.events`
   - `https://www.googleapis.com/auth/calendar.readonly`
4. Create OAuth Client ID with type `Web application`.
5. Add authorized redirect URIs:
   - `https://agenda.example.com/api/integrations/googlecalendar/callback`
   - `https://agenda.example.com/api/auth/callback/google`
6. Download the OAuth client JSON.
7. Paste the entire JSON string into `GOOGLE_API_CREDENTIALS`.
8. Keep `GOOGLE_LOGIN_ENABLED=false` unless you intentionally want Google login as an auth method.

After deploy, log into Cal.com as a seller and connect Google Calendar from the calendar settings.

## Microsoft/Outlook OAuth

In Azure App Registration:

1. Create a new app registration.
2. Choose multitenant if sellers may use different Microsoft tenants.
3. Configure web redirect URI:
   - `https://agenda.example.com/api/integrations/office365calendar/callback`
4. Copy the Application Client ID into `MS_GRAPH_CLIENT_ID`.
5. Create a client secret and set `MS_GRAPH_CLIENT_SECRET`.
6. Redeploy after adding Microsoft variables.

## API Integration Model

For the MVP, use normal Cal.com users for each seller unless managed users are explicitly validated in this self-hosted build.

Recommended MVP flow:

- Create one Cal.com user per seller.
- Each seller connects their Google/Microsoft calendar once.
- Leaderei stores the Cal.com `userId`, username and event type IDs for each seller.
- Leaderei calls API v2 for slots, create booking, cancel booking and reschedule booking.
- Leaderei receives booking lifecycle events through Cal.com webhooks.

Managed users and Platform OAuth exist in the codebase, but validate them in staging before making them the foundation of Leaderei onboarding. The current public Cal.com API docs say Platform-related access is in maintenance/deprecated status for new signups, so do not assume commercial Platform semantics without a self-hosted staging proof.

## Webhooks

Create webhooks after the API and first seller/event type are working.

For Leaderei, subscribe at minimum to:

- `BOOKING_CREATED`
- `BOOKING_CANCELLED`
- `BOOKING_RESCHEDULED`
- `BOOKING_REJECTED`
- `BOOKING_REQUESTED`, only if using confirmation-required events

Use an HTTPS Leaderei endpoint in production, for example:

```text
https://app.leaderei.com/api/webhooks/calcom
```

Store the Cal.com webhook secret in Leaderei and verify incoming signatures before processing payloads. Make webhook processing idempotent using the booking UID plus trigger event.

## First Deploy Checklist

1. Deploy staging with real HTTPS domains.
2. Confirm `calcom` opens at `https://agenda.example.com`.
3. Complete first-run setup and create the admin user.
4. Configure SMTP and send a test booking email.
5. Connect Google Calendar for one seller.
6. Create one event type for that seller.
7. Fetch available slots through API v2.
8. Create a booking through API v2.
9. Confirm the booking appears in Google Calendar.
10. Cancel and reschedule through API v2.
11. Create a Leaderei webhook subscription.
12. Verify Leaderei receives booking created, cancelled and rescheduled events.
13. Create the remaining seller users.
14. Set `NEXT_PUBLIC_DISABLE_SIGNUP=true` and redeploy after users are created.
15. Configure scheduled Postgres backups in Dokploy.

## Smoke Test Endpoints

Use the API v2 domain:

```bash
curl https://agenda-api.example.com/api/v2
```

After creating an API key in Cal.com settings:

```bash
curl https://agenda-api.example.com/api/v2/me \
  -H "Authorization: Bearer cal_<token>"
```

For OAuth-based API access, validate tokens with the same `/me` endpoint using `Authorization: Bearer <access-token>`.

## Operations

- Do not auto-deploy production from upstream `main`.
- Promote releases through staging first.
- Back up Postgres before upgrades.
- Keep `CALENDSO_ENCRYPTION_KEY` unchanged across deploys and restores.
- Monitor `calcom`, `calcom-api`, `redis` and Postgres logs after every deploy.
- Keep public signup disabled after provisioning admin and seller accounts.

Recommended update flow:

```text
upstream release/tag -> PR into fork -> staging build -> smoke test -> production deploy
```

Keep `origin` as `marioschiavon/s7-cal.com` and `upstream` as `calcom/cal.diy`.

## Known Risks

- Cal.diy public docs describe the community edition as self-hosted and recommend it mainly for personal/non-production use.
- API v2 is available, but managed users and Platform-style onboarding must be validated in this exact self-hosted deployment.
- Team, organization and managed-event semantics may differ from Cal.com Cloud or enterprise/on-prem offerings.
- OAuth callback domains must match exactly; changing `CALCOM_HOST` after build requires rebuilding the image.
- Webhook delivery should be treated as at-least-once; Leaderei must deduplicate events.
