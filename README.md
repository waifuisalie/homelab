# Homelab

A home server setup managed via Docker, built to automate my personal media and digital library management while running a dedicated observability stack to keep an eye on host resources and container health.

## What's running

- **Media & storage stack:** Nextcloud and MariaDB for file storage, Plex for media streaming, the automated download and management pipeline (Sonarr, Radarr, Readarr, Prowlarr, Bazarr, qBittorrent, FlareSolverr, Unpackerr), and Calibre with Calibre-Web for e-books.
- **Monitoring stack:** Prometheus, Grafana, Loki, OpenTelemetry Collector Contrib, node_exporter, and cAdvisor.

## Monitoring design decisions

A 60-second scrape interval was chosen over the default 15-second interval because high-frequency polling felt unnecessary for a non-critical homelab. 

Prometheus explicitly binds to `127.0.0.1:9091` rather than `0.0.0.0`, to make sure the raw metrics endpoints are never exposed across the local network. Grafana queries Prometheus directly over the isolated Docker internal `monitoring` bridge network, centralizing authentication through Grafana itself.

The OpenTelemetry Collector runs as root (`user: "0:0"`) with `/var/lib/docker/containers` mounted read-only. It needs to run as root to be able to read Docker's log directories, which is okay for a personal homelab server. OTel's purpose is to read every other container's logs anyway, but next steps will be to mitigate this following the 'least privilege' idea. 

## The alert

The alert watches disk usage on the dedicated storage drive `/mnt/hdd2`, so when utilization crosses 98% for more than 5 minutes, it triggers a warning. That's to avoid a false-positive, and my disk is almost full, so it was appropriate to have an alert to let me know. 

## CI

The GitHub Actions workflow runs on push and pull request to validate `docker-compose.yml`, verify dashboard JSON syntax, and run official linters (`promtool` for Prometheus and alert rules, `loki -verify-config`, and `otelcol-contrib validate`): [.github/workflows/validate.yml](.github/workflows/validate.yml).
