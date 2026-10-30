# Notes for working in this sandbox (Base44)

Jellyfin server (.NET 10). `docker-compose.base44.yml` is the runbook:

```bash
docker compose -f docker-compose.base44.yml up -d --build
```

- `jellyfin` — the server, built and run **from the bind-mounted source** with
  `dotnet watch` (hot reload). Listens on container port 8096, published on host 3000.
- `webclient` — one-shot job that builds the [jellyfin-web](https://github.com/jellyfin/jellyfin-web)
  frontend from upstream `master` into the `jellyfin-web` volume (the web client is
  not in this repo; `master` is version 13.0.0, matching this checkout). It is
  skipped once `/web/index.html` exists. To rebuild it:
  `docker compose -f docker-compose.base44.yml rm -sf webclient && docker volume rm jellyfin-base44_jellyfin-web && docker compose -f docker-compose.base44.yml up -d webclient`
  (about 2–3 minutes; the server will not start until it exits successfully).

## Things that are easy to trip over

- **Build output lives outside the repo.** The compose service sets
  `ArtifactsPath=/artifacts` (a volume) so `obj/`/`bin/` never appear in the
  bind-mounted checkout — this repo only gitignores `bin/`. Keep it that way.
- **First start is slow.** The container runs `dotnet restore` before
  `dotnet watch`; with a cold `nuget-packages` volume the first compile of the
  solution takes a few minutes (healthcheck `start_period` is 600s). Later starts
  are ~30s.
- **Persistent state** is in named volumes: `jellyfin-data` (`/data`: config,
  database, metadata, logs), `data-protection-keys` (keeps logins valid across
  restarts), `jellyfin-web`, `nuget-packages`, `build-artifacts`. Delete
  `jellyfin-data` to run the first-run setup wizard again.
- **No external credentials are needed to boot.** Metadata providers (TMDB, etc.)
  are configured in the web UI afterwards; ffmpeg comes from the distro
  (`ffmpeg 6.1`, not jellyfin-ffmpeg), so some transcoding features report
  missing filters — that is expected here.
- `--nonetchange` is passed because network-change detection is meaningless in a
  container. `ASPNETCORE_ENVIRONMENT=Development` is set for local diagnostics.
- Root `/` answers `302 -> /web/` (`BaseUrlRedirectionMiddleware` +
  `defaultredirect`); the web client is served under `/web`.

## Verify it works

```bash
curl -I http://localhost:3000/                                  # 302 -> /web/
curl -s http://localhost:3000/health                            # Healthy
curl -s http://localhost:3000/System/Info/Public                # Version 13.0.0
```

On a fresh `jellyfin-data` volume the browser shows the first-run setup wizard
(`"StartupWizardCompleted":false`). API docs: `/api-docs/swagger/index.html`.

## Tests (run inside the container — the host has no .NET runtime)

```bash
docker compose -f docker-compose.base44.yml exec -T jellyfin dotnet test tests/Jellyfin.Server.Tests/Jellyfin.Server.Tests.csproj
docker compose -f docker-compose.base44.yml exec -T jellyfin dotnet test tests/Jellyfin.Api.Tests/Jellyfin.Api.Tests.csproj
```
