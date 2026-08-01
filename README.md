# YOM

YOM means Your Own Messenger. This repository contains a compact Android messenger and a self-hosted Java server. It has one English interface, one black-and-white theme, private notification summaries, direct conversations, text messages, local message storage, delivery status, automatic token refresh, and WebSocket reconnect.

The app connects only to the server selected by the user. A server address is verified through `GET /api/server/info` before it is saved. Changing the server closes the WebSocket, removes both tokens, clears local account data, and returns to sign in.

## Repository layout

- `server` — Java 21 and Spring Boot server
- `android-app` — Android client written in Kotlin and Compose
- `.env.example` — safe environment variable template
- `docker-compose.yml` — optional server and PostgreSQL deployment

## Security model

Passwords are hashed with BCrypt. Access tokens are short-lived and refresh tokens expire independently. Message bodies are encrypted at rest while waiting for delivery and are deleted from the server queue after recipient acknowledgement. Delivery metadata is retained temporarily for retry safety.

YOM does not claim end-to-end encryption. The server processes message plaintext before encrypting the delivery queue. Use TLS, protect the host, keep PostgreSQL private, rotate secrets, and maintain backups.

Never commit `.env`, database files, private keys, tunnel tokens, tunnel credential JSON files, real user data, or signed release keystores.

## Server requirements

- Linux server, Ubuntu 24.04 or another current distribution recommended
- Java 21
- PostgreSQL 16 or newer
- 1 GB RAM minimum
- A domain managed by Cloudflare if Cloudflare Tunnel is used

Docker and Docker Compose are optional.

## Generate secrets

Create a private environment file from the example:

```bash
cp .env.example .env
```

Generate independent values:

```bash
openssl rand -hex 32
openssl rand -base64 32
openssl rand -base64 36
```

Use the first value for `APP_JWT_SECRET`, the second for `APP_CRYPTO_KEY`, and the third for `POSTGRES_PASSWORD`. Do not reuse them.

## Quick start with Docker

Edit `.env`, then run from the repository root:

```bash
docker compose up --build -d
docker compose logs -f server
```

PostgreSQL is available only to the Compose network. The messenger server listens on port `8080`.

Verify it locally:

```bash
curl http://127.0.0.1:8080/api/server/info
```

Expected structure:

```json
{
  "name": "My Y.O. Messenger Server",
  "version": "1.0.0",
  "apiVersion": 1
}
```

## PostgreSQL without Docker

Install PostgreSQL and open its console:

```bash
sudo apt update
sudo apt install postgresql
sudo -u postgres psql
```

Create a role and database. Replace the example password:

```sql
CREATE USER yo_messenger WITH PASSWORD 'replace-with-your-password';
CREATE DATABASE yo_messenger OWNER yo_messenger;
\q
```

Flyway creates and upgrades the schema automatically when the server starts. Do not run migration files manually.

## Server environment variables

| Variable | Required | Purpose |
| --- | --- | --- |
| `SPRING_DATASOURCE_URL` | yes | PostgreSQL JDBC URL |
| `SPRING_DATASOURCE_USERNAME` | yes | PostgreSQL role |
| `SPRING_DATASOURCE_PASSWORD` | yes | PostgreSQL password |
| `APP_JWT_SECRET` | yes | JWT signing secret with at least 32 characters |
| `APP_CRYPTO_KEY` | yes | Base64-encoded 32-byte AES key |
| `APP_CORS_ORIGINS` | no | Comma-separated origin patterns; default is `*` |
| `APP_SERVER_NAME` | no | Name returned to the Android app |
| `APP_ACCESS_TOKEN_MINUTES` | no | Access token lifetime; default `15` |
| `APP_REFRESH_TOKEN_DAYS` | no | Refresh token lifetime; default `30` |
| `SERVER_PORT` | no | HTTP port; default `8080` |

## Build and run the JAR

Install Java 21 and confirm it is active:

```bash
java -version
```

Build the server:

```bash
cd server
./gradlew clean test bootJar
```

The JAR is created at `server/build/libs/yo-messenger-server-1.0.0.jar`.

Export the required variables and start it:

```bash
export SPRING_DATASOURCE_URL='jdbc:postgresql://127.0.0.1:5432/yo_messenger'
export SPRING_DATASOURCE_USERNAME='yo_messenger'
export SPRING_DATASOURCE_PASSWORD='replace-with-your-password'
export APP_JWT_SECRET='replace-with-your-generated-secret'
export APP_CRYPTO_KEY='replace-with-your-generated-base64-key'
export APP_SERVER_NAME='My Y.O. Messenger Server'
java -jar build/libs/yo-messenger-server-1.0.0.jar
```

## Run with systemd

Create a dedicated user and directories:

```bash
sudo useradd --system --home /opt/yo-messenger --shell /usr/sbin/nologin yomessenger
sudo mkdir -p /opt/yo-messenger /etc/yo-messenger
sudo cp server/build/libs/yo-messenger-server-1.0.0.jar /opt/yo-messenger/server.jar
sudo chown -R yomessenger:yomessenger /opt/yo-messenger
```

Create `/etc/yo-messenger/server.env`:

```text
SPRING_DATASOURCE_URL=jdbc:postgresql://127.0.0.1:5432/yo_messenger
SPRING_DATASOURCE_USERNAME=yo_messenger
SPRING_DATASOURCE_PASSWORD=replace-with-your-password
APP_JWT_SECRET=replace-with-your-generated-secret
APP_CRYPTO_KEY=replace-with-your-generated-base64-key
APP_CORS_ORIGINS=*
APP_SERVER_NAME="My Y.O. Messenger Server"
APP_ACCESS_TOKEN_MINUTES=15
APP_REFRESH_TOKEN_DAYS=30
SERVER_PORT=8080
```

Protect it:

```bash
sudo chown root:yomessenger /etc/yo-messenger/server.env
sudo chmod 640 /etc/yo-messenger/server.env
```

Create `/etc/systemd/system/yo-messenger.service`:

```ini
[Unit]
Description=Y.O. Messenger Server
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
Type=simple
User=yomessenger
Group=yomessenger
WorkingDirectory=/opt/yo-messenger
EnvironmentFile=/etc/yo-messenger/server.env
ExecStart=/usr/bin/java -jar /opt/yo-messenger/server.jar
Restart=on-failure
RestartSec=5
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true

[Install]
WantedBy=multi-user.target
```

Enable and inspect the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now yo-messenger
sudo systemctl status yo-messenger
journalctl -u yo-messenger -f
```

## Cloudflare Tunnel and your domain

The simplest supported setup is a remotely managed tunnel:

1. Add your domain to Cloudflare.
2. In the Cloudflare dashboard, open **Networking → Tunnels** and create a tunnel.
3. Select Linux and run the installation command generated by Cloudflare on the messenger server. The command contains a private tunnel token; never save it in this repository.
4. Add a **Published application** route.
5. Set the hostname to a subdomain such as `chat.example.com`.
6. Set the service URL to `http://localhost:8080`.
7. Confirm that the connector and route are healthy.

Cloudflare terminates HTTPS and proxies normal HTTP and WebSocket traffic to the same local service. No separate `/ws` tunnel rule is needed.

Verify the public endpoint:

```bash
curl https://chat.example.com/api/server/info
```

The current Cloudflare instructions are available in the [Tunnel setup guide](https://developers.cloudflare.com/tunnel/setup/) and [Linux service guide](https://developers.cloudflare.com/tunnel/advanced/local-management/as-a-service/linux/).

## Build the Android app

Install Android Studio, JDK 21, Android SDK Platform 36, and the matching SDK Build Tools.

Open the `android-app` directory in Android Studio and select JDK 21 as the Gradle JDK. To build from a terminal:

```bash
cd android-app
./gradlew clean testDebugUnitTest assembleDebug
```

On Windows:

```powershell
cd android-app
.\gradlew.bat clean testDebugUnitTest assembleDebug
```

The debug APK is created at `android-app/app/build/outputs/apk/debug/app-debug.apk`.

For a distributable release, use **Build → Generate Signed App Bundle or APK** in Android Studio. Store the release keystore outside the repository.

## Connect the app to your server

1. Install the APK.
2. Open YOM.
3. Tap **Your server** at the top.
4. Enter the base address only, for example `https://chat.example.com`.
5. Tap **Check and save**.
6. Register a username and password.

For `https://chat.example.com`, the app uses:

- HTTP API: `https://chat.example.com/api/`
- WebSocket: `wss://chat.example.com/ws`

For `http://192.168.1.12:8080`, it uses:

- HTTP API: `http://192.168.1.12:8080/api/`
- WebSocket: `ws://192.168.1.12:8080/ws`

The app permits cleartext HTTP so local LAN servers can work, but HTTP exposes credentials and messages to the network. Use HTTPS for every public deployment.

## Backups and updates

Back up PostgreSQL regularly:

```bash
pg_dump -Fc yo_messenger > yo-messenger-$(date +%F).dump
```

Before updating, back up the database and the server environment file. Replace the JAR, then restart:

```bash
sudo systemctl restart yo-messenger
```

Flyway applies new migrations at startup. Never edit a migration that has already run on a deployed database.

## License

This project is available under the MIT License. See `LICENSE`.
