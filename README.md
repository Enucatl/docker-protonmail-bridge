# Proton Mail Bridge deployment

**Image building has moved to [Enucatl/proton-bridge](https://github.com/Enucatl/proton-bridge),
which contains our minimalistic, headless Proton Mail Bridge implementation.**
This repository now contains only its deployment configuration and instructions.

This repository deploys the published Linux amd64 image from
[Enucatl/proton-bridge](https://github.com/Enucatl/proton-bridge). Image builds,
updates, tests, and vulnerability scans belong there. Compose tracks the
`latest` tag. See the
[image operation instructions](https://github.com/Enucatl/proton-bridge/blob/main/HEADLESS.md).

Run commands from `/opt/docker/protonmail-bridge`. The sibling
[compose-security-baseline](https://github.com/Enucatl/docker-compose-security-baseline)
provides the read-only root filesystem, dropped capabilities, no-new-privileges,
and resource limits.

## State, key, and certificates

| Container path | Source | Purpose |
| --- | --- | --- |
| `/data` | Named volume `protonmail-bridge_bridge-state` | Private writable state, owned by UID/GID 1000:1000. Docker initializes a fresh volume from the image. |
| `/run/secrets/bridge_vault_key` | Read-only Compose secret `./secrets/vault_key` | Exactly 32 raw bytes that unlock the vault; kept outside the state volume. |
| `/tmp` | Private 64 MiB tmpfs | Temporary files, owned by UID/GID 1000:1000. |
| `/protonmail/certs/cert.pem` | Traefik `docker_fullchain.pem`, read-only | Existing TLS certificate chain. |
| `/protonmail/certs/key.pem` | Traefik `docker_key.pem`, read-only | Matching TLS private key, readable by the remapped service UID. |

For **fresh state only**, create the key without overwriting an existing file:

```sh
mkdir -p -m 0700 secrets
(umask 077; set -C; openssl rand 32 > secrets/vault_key)
```

Puppet's `data/nodes/docker.home.arpa.yaml` manages secret access for host UID
101000 (Docker remap base 100000 + service UID 1000). Apply its configuration
before startup. To grant the same access immediately on this host:

```sh
setfacl -m u:101000:--x,g::---,o::--- secrets
setfacl -m u:101000:r--,g::---,o::--- secrets/vault_key
```

The file reports mode 0640 because the group bits represent the named ACL mask;
the owning group has no access. Compose file secrets preserve host permissions
and ACLs. Missing, malformed, or unsafe keys stop startup. Never replace the key
for existing state. Back up the key separately from the full state volume and
encrypt both backups; SQLite metadata and logs are plaintext.

Bridge loads the mounted certificates directly; no `cert import` step is needed.
Restart after certificate renewal. If both certificate mounts are removed,
Bridge generates and persists a self-signed certificate instead.

## First login and sync

The new `bridge-state` volume starts empty. The previous pass/GPG volume,
`protonmail-bridge_protonmail`, remains untouched. A fresh login downloads the
mailbox again and creates new local Bridge credentials and IMAP identities.
Keep the old volume and image until mailbox/client acceptance passes.

```sh
docker compose pull
docker compose down
docker compose run --rm --no-deps protonmail-bridge --cli --vault-key-file /run/secrets/bridge_vault_key
```

The initial `down` preserves the old volume and lets Compose recreate the
network with IPv6 enabled. In the CLI, run `login`, complete password/2FA/mailbox-password prompts, then
`info` to read the local Bridge credentials and `exit`. Enter credentials only
in the interactive CLI. Only one Bridge process can use the state volume;
always stop the service before opening the CLI.

```sh
docker compose up -d --wait --wait-timeout 120
docker compose logs --tail=100 -f protonmail-bridge
```

The image supplies its own healthcheck, which verifies TLS and both mail
greetings. Healthy status does not confirm login or completed synchronization.
Verify initial sync, folders, flags, attachments, sending, and reconnect after
a restart with a real client.

## Mail clients

Both protocols require **SSL/TLS (implicit TLS)**. Change previous STARTTLS
settings and use the credentials from `info`, rather than the Proton password.

| Protocol | Host port | Container port |
| --- | --- | --- |
| IMAP | 10243 | 1143 |
| SMTP | 10125 | 1025 |

The host publishes ports 10243 (IMAP) and 10125 (SMTP) for clients, including
containers such as Paperless. Connect to the certificate-covered hostname
`bridge.docker.home.arpa` and trust the certificate issuer. Both use implicit TLS.
The dedicated default network allows outbound Proton HTTPS and enables IPv6.

## Updates and recovery

Change the image pin in `docker-compose.yml`, then run `docker compose pull`
and `docker compose up -d --wait --wait-timeout 120`. The runtime has no automatic
updater, shell, pass/GPG, or socat forwarding.

`docker compose down` preserves state; `down -v` deletes the new state volume.
For migration that preserves the old Bridge passwords and IMAP identities,
follow the image's [migration instructions](https://github.com/Enucatl/proton-bridge/blob/main/HEADLESS.md#migration-and-release-acceptance)
instead of creating fresh state. Export the existing vault key and snapshot
all old state before migration. Image-only rollback may require restoring that
snapshot and logging in again.
