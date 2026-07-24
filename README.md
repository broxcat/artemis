# Artemis

A self-hosted [Velociraptor](https://docs.velociraptor.app) homelab, run via Docker Compose behind an
nginx reverse proxy with HTTPS. Named after Artemis, goddess of the hunt — fitting for a DFIR / threat
hunting platform.

> **Homelab disclaimer:** This setup uses a self-signed TLS certificate and default single-node
> configuration. It is intended for local learning, testing, and threat-hunting practice — **not**
> production or internet-exposed use.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/)
- OpenSSL (for generating the self-signed certificate)

## Installation

1. Clone the repo:

   ```bash
   git clone https://github.com/broxcat/artemis.git
   cd artemis
   ```

2. Copy the sample environment file and fill in your own values:

   ```bash
   cp .env.sample .env
   ```

3. Generate a strong password for `VELOX_PASSWORD` in `.env`:

   ```bash
   openssl rand -base64 24
   ```

4. Generate the self-signed TLS certificate used by nginx:

   ```bash
   mkdir -p certs
   openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
     -keyout certs/key.pem -out certs/cert.pem \
     -subj "/CN=localhost"
   ```

5. Start the stack:

   ```bash
   docker compose up -d
   ```

6. Access the Velociraptor GUI at **https://localhost** (accept the self-signed certificate warning),
   and log in with `VELOX_USER` / `VELOX_PASSWORD` from your `.env`.

### Two separate URLs — don't confuse them

- **`VELOX_SERVER_URL`** (`https://localhost:8000/` by default) is the URL Velociraptor **agents** use to
  phone home to the server on the frontend port (8000). This is baked into every repacked agent binary and
  is **not** proxied by nginx.
- **The GUI URL** (`https://localhost` on port 443) is what **you**, the operator, browse to. It's served
  by nginx, which reverse-proxies to the Velociraptor GUI on its internal port 8889.

These two are independent — changing one does not affect the other.

## Deploying the Velociraptor Agent

Agent binaries are repacked with the server's config directly inside the running `velociraptor`
container. Repacked binaries embed the server's CA and connection details, so **never commit them** (see
[Security notes](#security-notes)).

### Windows

1. Repack the Windows client from inside the container:

   ```bash
   docker exec -it velociraptor ./velociraptor config repack \
     --exe /opt/velociraptor/windows/velociraptor_client.exe \
     client.config.yaml \
     clients/windows/agent.exe
   ```

2. Copy the repacked binary out to the host:

   ```bash
   docker cp velociraptor:/velociraptor/clients/windows/agent.exe ./agent.exe
   ```

3. Run it manually to test:

   ```powershell
   .\agent.exe client -v
   ```

4. Install it as a permanent Windows service (requires an **elevated** PowerShell prompt):

   ```powershell
   .\agent.exe service install
   ```

### Linux

1. Repack the Linux client from inside the container:

   ```bash
   docker exec -it velociraptor ./velociraptor config repack \
     --exe /opt/velociraptor/linux/velociraptor_client \
     client.config.yaml \
     clients/linux/agent
   ```

2. Copy it out and make it executable:

   ```bash
   docker cp velociraptor:/velociraptor/clients/linux/agent ./agent
   chmod +x agent
   ```

3. Run it manually to test:

   ```bash
   ./agent client -v
   ```

4. Install it as a systemd service (Velociraptor's `service install` subcommand uses systemd on Linux
   under the hood; run as root):

   ```bash
   sudo ./agent service install --config client.config.yaml -v
   ```

### macOS

1. Repack the macOS client from inside the container:

   ```bash
   docker exec -it velociraptor ./velociraptor config repack \
     --exe /opt/velociraptor/mac/velociraptor_client \
     client.config.yaml \
     clients/mac/agent
   ```

2. Copy it out and make it executable:

   ```bash
   docker cp velociraptor:/velociraptor/clients/mac/agent ./agent
   chmod +x agent
   ```

3. Run it manually to test:

   ```bash
   ./agent client -v
   ```

4. Install it as a persistent service. Velociraptor's `service install` subcommand also supports macOS,
   registering itself via `launchd` — run it with root privileges:

   ```bash
   sudo ./agent service install --config client.config.yaml -v
   ```

   If this doesn't behave as expected on your macOS version, refer to the official docs for the current
   launchd-based instructions: https://docs.velociraptor.app/docs/deployment/clients/

## Security notes

- **Never commit `.env`, `certs/`, or repacked agent binaries.** They contain credentials, private keys,
  and embedded server configuration/CA material. All are excluded via `.gitignore`.
- The self-signed certificate generated above is fine for a local homelab, but browsers and agents will
  not trust it by default — that's expected and not a bug.
- `VELOX_SERVER_URL` must be a hostname or IP that every agent OS can actually resolve and reach on port
  8000. Using `localhost` only works if the agent runs on the same host as the server — if you deploy
  agents on other machines/VMs, change `VELOX_SERVER_URL` and `VELOX_FRONTEND_HOSTNAME` to the server's
  real reachable IP or DNS name before repacking clients, otherwise the agent will fail to check in.

## Credits

- [Velociraptor documentation](https://docs.velociraptor.app)
- [Velocidex/velociraptor](https://github.com/Velocidex/velociraptor) — the official project this homelab
  is built on
