# Development image for the Jellyfin server used by docker-compose.base44.yml.
# Plain SDK base image: the application source is bind-mounted at /src at runtime,
# never baked in, so edits are picked up by `dotnet watch` without a rebuild.
FROM mcr.microsoft.com/dotnet/sdk:10.0-noble

ENV DOTNET_CLI_TELEMETRY_OPTOUT=1 \
    DOTNET_NOLOGO=1 \
    DOTNET_SKIP_FIRST_TIME_EXPERIENCE=1

# ffmpeg is needed for media probing/transcoding; fontconfig/libfreetype for SkiaSharp
# image processing; curl for the compose healthcheck.
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        ffmpeg \
        libfontconfig1 \
        libfreetype6 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /src
