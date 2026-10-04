# Setup and recovery

## Windows wizard

Extract the release ZIP into a permanent folder, start Docker Desktop with its Linux engine, and double-click `Start Setup.cmd`. Setup uses Windows PowerShell and does not require Node on the host. If Docker is missing, click **Install Docker Desktop**. This opt-in helper downloads from Docker, verifies its Authenticode signature/publisher, and opens the interactive installer. Review Docker’s terms and Windows prompts. Docker may require virtualization/WSL configuration or a restart; the wizard will not make those changes for you.

Choose an unused port (default 8090), a household passcode of at least eight characters, and a four-digit Owner PIN. Optionally enter a TMDB **API Read Access Token** and your Plex server URL/token. Inputs are masked; setup stores credentials in `.env`, restricts that file to the current Windows account and SYSTEM, and does not print them. Remember your passcode/PIN or store them in your password manager.

The first build can take several minutes. Build progress is saved in `setup-build.log` and `setup-error.log`. The wizard only opens the app once `/api/health` responds successfully. Re-running it preserves an existing `.env` and database. It refuses to pair new credentials with an existing `plexrentals_data` volume when the original `.env` is missing.

The default installation creates only the `plexrentals` Compose project and its `plexrentals_data` volume. It binds to **127.0.0.1**, not your LAN or the public Internet. It has no access to your NAS or host media folders.

## Optional guided service stack

After the main app is healthy, click **Next: guided Sonarr / Radarr / Prowlarr setup**.

1. Choose a service username and 12+ character password. Choose an **empty local fixed-drive folder**. Existing libraries and network-drive mappings are intentionally not adopted.
2. Optionally enter an existing qBittorrent Web UI address, login, and reported download path. That path must correspond to the selected folder's downloads subfolder; a mapping cannot move files between unrelated locations. For a client on the same Windows PC use host.docker.internal.
3. Optionally enter your authorized Torznab endpoint/key. For other providers, use Prowlarr's indexer wizard afterward.
4. Check the explicit permission box and click **Set it up for me**. Setup creates new service config volumes and generated API keys, enforces Forms login, sets HD-1080p profiles and library roots, links the apps, and tests any supplied connections.
5. Read setup-services.log for the finishing checklist. Check qBittorrent category save paths, enable the tested download clients and indexers, review quality/size limits, and only then enable acquisition in Plex Rentals. No acquisition test runs automatically.

| Host folder | Sonarr/Radarr path |
|---|---|
| Chosen folder/library/movies | /data/library/movies |
| Chosen folder/library/tv | /data/library/tv |
| Chosen folder/downloads | /data/downloads |

Service pages: [Sonarr](http://localhost:18989), [Radarr](http://localhost:17878), [Prowlarr](http://localhost:19696). All ports bind to loopback. Plex must be configured to scan the resulting movie/TV folders; Plex on another machine needs separately planned shared storage. This wizard does not move existing media or configure NAS permissions.

The private .setup-private.json file holds the entered service credentials with a restricted Windows ACL. Keep it private alongside .env; neither is copied into the Docker image or public repository. On retry, authentication checkpoints preserve service logins. You can correct connection details that have not yet been saved successfully. Already-created resources are preserved; edit them in their service UI afterward. Concurrent setup runs are unsupported.

When the service stack is selected, use both Compose files for manual commands:

```sh
docker compose -f compose.yaml -f setup/stack.compose.yaml up -d --build --wait
docker compose -f compose.yaml -f setup/stack.compose.yaml stop
docker compose -f compose.yaml -f setup/stack.compose.yaml ps
```

Rerunning the main wizard automatically includes the service overlay. Preserve all four data/config volumes during updates. Back up service configurations as well as the app database.

## First sign-in and connection checklist

1. Enter the household passcode, choose **Owner**, and enter its PIN. Rename Owner and family profiles in Settings. Existing profiles are never renamed by setup.
2. **TMDB:** if you skipped the token, stop the app and add `TMDB_READ_TOKEN=your-token` to the private `.env`, then start again. Get the Read Access Token from your own TMDB account's API settings. Never put it in a browser-side `VITE_` variable.
3. **Plex:** stop the app, add `PLEX_URL=http://your-plex-host:32400` and `PLEX_TOKEN=your-own-token` to `.env`, then start again. For Plex on the same PC, a Docker container normally reaches it using `host.docker.internal`, not `localhost`. Use a server address reachable from the container. Confirm library sync and availability before enabling anything else.
4. **Optional request automation:** enter your own Sonarr/Radarr API keys through **Settings → Integrations**. Test each connection. Container service names only resolve on a shared Docker network; do not type those names into your host browser. Configure root folders, download-client categories, and consistent paths separately. Dispatch starts only after you explicitly enable it.
5. **Downloads/indexers:** the optional guided service setup below installs Sonarr, Radarr and Prowlarr. It can connect an existing qBittorrent client and an authorized Torznab source. Other providers/accounts require their own setup. Set quality and size limits there. The distribution does not inherit another household’s 8 GB setting.
6. **Storage review:** optional and disabled without explicit read-only mounts and `STORAGE_INVENTORY_ENABLED=1`. Map movies to `/inventory/movies` and TV to `/inventory/tv`; never mount unrelated folders. Validate NAS failure behavior before enabling imports.
7. **Remote access:** not configured by this installer. Use HTTPS and a suitable access policy before exposing the app. Keep Sonarr/Radarr, Docker, and NAS administration private. Do not expose the four-digit PIN screen as your only public access control.

The daily autoapproval allowance is per app profile, not a verified email identity; its reset timezone is currently America/Chicago. Do not use shared household access as a multi-tenant service.

## Start, stop, and update

Run these commands in the installation folder:

```sh
docker compose -f compose.yaml up -d --build --wait
docker compose -f compose.yaml stop
docker compose -f compose.yaml ps
```

After a Windows restart, sign in and start Docker Desktop (or enable its start-on-sign-in option). With Docker running, `restart: unless-stopped` restarts the app unless you explicitly stopped it. If you stopped it, run Setup again or the first command above. Windows being powered on without Docker running is not sufficient.

Before updating, export a backup through the app and keep a private copy of `.env`. Extract the new source package to a separate folder for review, then copy its source files over the existing installation, preserving `.env`. Run Setup again to rebuild. Do not run `down --volumes`: that would remove the database volume.

Backups generated inside the Docker volume are not protection against losing the disk. Copy a downloaded backup to another secure location. Restoring a backup requires the same deployment configuration; keep credentials private and confirm the database before turning integrations back on.

## Troubleshooting

- **Port already in use:** select another port before the first install. For an existing installation, stop it, edit `INSTALL_PORT` in `.env`, and start again.
- **Docker unavailable:** start Docker Desktop and confirm Linux containers. Follow [official Windows prerequisites](https://docs.docker.com/desktop/setup/install/windows-install/).
- **Build failed:** inspect the two setup log files. Check Internet access and disk space, then rerun Setup. Do not post credentials or raw private logs publicly.
- **Blank fictional preview:** no TMDB token is configured. The app does not download an actual catalog into the release package.
- **Existing volume but missing configuration:** restore the original `.env` or locate the original installation. Setup deliberately refuses to overwrite existing data.
- **Lost PIN:** use your retained configuration and documented recovery process; do not delete the database as a shortcut.

Advanced macOS/Linux users can use the same Compose file after privately creating `.env` with `INSTALL_PORT`, `HOUSEHOLD_PASSCODE`, `ADMIN_PIN`, and independently generated 32-byte random hex `SESSION_SECRET` and `PLEX_WEBHOOK_SECRET`. Apply file mode 600. The graphical wizard is Windows-only.
