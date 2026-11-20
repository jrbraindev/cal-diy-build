# Cal.diy ARM64 — Standalone runtime proof of concept

**Experimental only. Do not point the existing Coolify Cal.diy service or production database at this image.**

## Why
The deployed Cal.diy image has about 7.34 GB, including a 4.61 GB `COPY /calcom` layer. Runtime diagnostics show about 3.1 GB in `node_modules`, while the existing `apps/web/.next/standalone` tree is about 393 MB.

This proof of concept uses an immutable reference to the **currently running image**, not the upstream `calcom/cal.diy:main` branch (which has moved to Node 22). It extracts only the already-built Next.js standalone runtime, public assets and static assets. It publishes a new, run-specific tag to GHCR.

## Safety boundaries
- The existing `arm64` image tag and running Cal.diy container are untouched by the POC build.
- The POC intentionally **does not run** the original `scripts/start.sh`, `prisma migrate deploy`, `seed-app-store.ts`, or URL-placeholder replacement. Those must be preserved via a separate, properly tested init/migration mechanism before any production replacement.
- Do not test against the live production DB. Use a **separately restored test DB**, with non-production credentials, isolated from real calendars, webhooks, SMTP and Teams/Google integrations.
- The built `NEXT_PUBLIC_WEBAPP_URL` is still the original production hostname; this limits functional verification until the runtime URL-rewrite step is safely replaced.
- Native libraries and runtime requirements may be missing from `node:20-bookworm-slim`. Test before considering this deployable.
- If the image build fails due to source GHCR package visibility, confirm GitHub Actions has `packages: read` access to the source package; do not make private images public just to get a build to work.
- Never delete the current image or its rollback copy during this experiment.

## How to build
Run the existing **Build Cal.diy ARM64** GitHub Actions workflow manually, selecting:
- Branch: `experiment/runtime-poc-20261120`
- Variant: `runtime-poc`

The experimental image will be tagged `ghcr.io/jrbraindev/cal-diy-build:arm64-runtime-poc-<run_number>`. No deployment is performed by this workflow.

## Acceptance checks (required before production migration)
1. Image size compared with 7.34 GB baseline.
2. Verify all native modules and dynamic imports under the standalone server can start.
3. Healthchecks, login, scheduling, create/change/cancel booking, timezones.
4. Test migrations and seeding as isolated idempotent startup jobs on restored test DB.
5. Google Calendar, Teams, SMTP and webhooks with safe test credentials.
6. Automated rollback plan with immutable image digest; retain old image until everything is verified.

## Next engineering phase
Provide a separate, *minimal* migrator/init image or job carrying just Prisma and seeding dependencies, and an equivalent URL placeholder replacement strategy. Until then, the runtime image is strictly for technical feasibility tests.
