# What is this fork about?
[![image](https://img.shields.io/badge/image-ghcr.io%2Fenucatl%2Fprotonmail--bridge-2496ED?logo=docker&logoColor=white)](https://github.com/Enucatl/protonmail-bridge-docker/pkgs/container/protonmail-bridge)
[![latest tag](https://img.shields.io/github/v/release/ProtonMail/proton-bridge?label=latest&color=2496ED)](https://github.com/Enucatl/protonmail-bridge-docker/pkgs/container/protonmail-bridge)
[![image size](https://img.shields.io/badge/image%20size-~141%20MB-2496ED)](https://github.com/Enucatl/protonmail-bridge-docker/blob/main/README.md)
[![downloads](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fghcr-badge.elias.eu.org%2Fapi%2FEnucatl%2Fprotonmail-bridge-docker%2Fprotonmail-bridge&query=downloadCount&label=docker%20pulls&color=2496ED&logo=docker&logoColor=white)](https://github.com/Enucatl/protonmail-bridge-docker/pkgs/container/protonmail-bridge)
[![build](https://img.shields.io/github/actions/workflow/status/Enucatl/protonmail-bridge-docker/build.yaml?branch=main&label=build)](https://github.com/Enucatl/protonmail-bridge-docker/actions/workflows/build.yaml)
[![scan](https://img.shields.io/badge/scan-Trivy-1904DA?logo=trivy&logoColor=white)](https://github.com/Enucatl/protonmail-bridge-docker/actions/workflows/build.yaml)
[![security](https://img.shields.io/badge/vulnerabilities-GitHub%20Security-2EA44F?logo=github&logoColor=white)](https://github.com/Enucatl/protonmail-bridge-docker/security/code-scanning)


- Run Bridge noninteractively in steady state, while keeping `init` interactive
- Minimize the runtime image by shipping only the headless Bridge binary plus required runtime dependencies
- Reduce the image from 194 MB by removing the GUI launcher and `vault-editor`, stripping the shipped binary, and using a slimmer runtime base image
- Harden the default runtime by dropping Linux capabilities and enabling `no-new-privileges`
- Add Trivy image vulnerability scanning to catch packaged dependency issues
- Keep `pass` and GPG in the image intentionally: we checked the upstream Linux keychain code and `pass` is still the most practical self-contained backend for a container
- Keep `socat` intentionally: Bridge is still hardcoded to listen on `127.0.0.1`, so the container needs a forwarding shim to expose IMAP and SMTP externally
- Allow connecting with ipv6
- Changing configurations in `docker-compose.yml` to use my external network
- Publish my image to github with actions, only for amd64
- Add a healthcheck


# ProtonMail IMAP/SMTP Bridge Docker Container

This is an unofficial Docker container of the [ProtonMail Bridge](https://protonmail.com/bridge/). Some of the scripts are based on [Hendrik Meyer's work](https://gitlab.com/T4cC0re/protonmail-bridge-docker).
Further developed by shexn here: [https://github.com/shenxn/protonmail-bridge-docker](https://github.com/shenxn/protonmail-bridge-docker).

## Initialization

Initialization is the only workflow that needs an interactive TTY. It creates the local `pass` store, starts the Bridge CLI, and lets you complete login and 2FA.

The Compose file includes a dedicated `protonmail-bridge-init` service behind the `init` profile. It mounts only `/data` plus the external TLS certificate files. The certs are mounted at `/protonmail/certs` and then imported into Bridge's vault on exit from the interactive CLI; Bridge does not automatically use `cert.pem` and `key.pem` just because they exist under the config directory.

```bash
docker compose build
docker compose --profile init run --rm protonmail-bridge-init
```

Wait for the bridge to startup, then you will see a prompt appear for [Proton Mail Bridge interactive shell](https://proton.me/support/bridge-cli-guide). Use the `login` command and follow the instructions to add your account into the bridge. Then use `info` to see the configuration information (username and password). After that, use `exit` to exit the bridge.

## Add custom certificates

When the CLI exits successfully, the entrypoint runs `cert import` automatically using `/protonmail/certs/cert.pem` and `/protonmail/certs/key.pem`, then stores those file paths in `vault.enc`.
If you add or rotate certificates after the initial setup, rerun:

```bash
docker compose stop protonmail-bridge
docker compose --profile init run --rm protonmail-bridge-init import-certs
docker compose up -d
```

## Run

After initialization, the service runs headlessly and no longer needs `tty` or `stdin_open`.

```bash
docker compose up -d
```

The container starts `/protonmail/bridge --noninteractive` and uses `socat` to expose IMAP and SMTP over IPv6-capable listeners because Bridge itself binds only to `127.0.0.1`.

## Runtime layout

- Persistent state lives under `/data`.
- Bridge config, cache, data, GPG home, and password store all live inside `/data`.
- The external TLS certificate and key are bind-mounted at `/protonmail/certs/{cert,key}.pem` and referenced from the Bridge vault after `cert import`.
- The example compose file drops all Linux capabilities and enables `no-new-privileges`.
- The published ports map host `10125 -> 1125` for SMTP and `10243 -> 1243` for IMAP.

## Kubernetes

If you want to run this image in a Kubernetes environment. You can use the [Helm](https://helm.sh/) chart (https://github.com/k8s-at-home/charts/tree/master/charts/stable/protonmail-bridge) created by [@Eagleman7](https://github.com/Eagleman7). More details can be found in [#23](https://github.com/shenxn/protonmail-bridge-docker/issues/23).

If you don't want to use Helm, you can also reference to the guide ([#6](https://github.com/shenxn/protonmail-bridge-docker/issues/6)) written by [@ghudgins](https://github.com/ghudgins).

## Bridge CLI Guide

The initialization step exposes the bridge CLI so you can do things like switch between combined and split mode, change proxy, etc. The [official guide](https://protonmail.com/support/knowledge-base/bridge-cli-guide/) gives more information on to use the CLI.

## Build

For anyone who wants to build this container on your own, the image is built from source in the `build/` directory and packages the Bridge release pinned in `build/VERSION`.

```bash
docker build -t protonmail-bridge ./build
```

The Dockerfile downloads the tagged Bridge release archive, builds the headless binary, strips it, and copies only the runtime artifacts into the final image.

## Security baseline

This compose project uses the shared [docker-compose-security-baseline](https://github.com/Enucatl/docker-compose-security-baseline) for common container hardening defaults, including capabilities, no-new-privileges, memory/swap, and PID limits.

## Trivy remediation

The repository includes an opt-in `workflow_run` remediation workflow. It uses Codex with OpenRouter's `deepseek/deepseek-v4.1-flash` model to propose fixes for fixed HIGH/CRITICAL image vulnerabilities. The paid Codex job is restricted to failed image builds from a push to `main`; pull-request builds still run normal CI and Trivy but never invoke Codex.

Set the repository Actions secret `OPENROUTER_API_KEY`. The workflow creates a remediation PR; it does not modify `main` directly. Remediation PRs receive a unique temporary image tag so the normal build workflow can rebuild and scan the proposed image before merge. See the [shared remediation workflow documentation](https://github.com/Enucatl/docker-compose-security-baseline#opt-in-trivy-remediation) for the workflow and security details.
