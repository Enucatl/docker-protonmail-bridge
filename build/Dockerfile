FROM golang:1.26.6-trixie AS build

ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
    && apt-get upgrade -y --no-install-recommends \
    && apt-get install -y --no-install-recommends build-essential ca-certificates curl libcbor-dev libfido2-dev libsecret-1-dev pkg-config \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build/
COPY build.sh VERSION /build/
RUN bash build.sh

FROM debian:13-slim

ARG DEBIAN_FRONTEND=noninteractive
ARG APT_CACHE_BUST

EXPOSE 1125/tcp
EXPOSE 1243/tcp

HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD timeout 5 bash -c 'exec 3<>/dev/tcp/localhost/1125 && exec 4<>/dev/tcp/localhost/1243 || exit 1' || exit 1

RUN test -n "${APT_CACHE_BUST}" \
    && apt-get update \
    && apt-get upgrade -y --no-install-recommends \
    # Runtime dependency rationale:
    # - bash: required by /protonmail/entrypoint.sh, and pass itself is a bash script.
    # - ca-certificates: Bridge talks to Proton over TLS and needs the system trust store.
    # - libfido2-1: the built bridge binary links to libfido2.so.1.
    # - libsecret-1-0: the built bridge binary links to libsecret-1.so.0 for Linux
    #   keychain support/probing, even when pass is the backend we actually use.
    # - gnupg: pass shells out to gpg at runtime; install this explicitly so the real
    #   runtime dependency is visible instead of being hidden as a transitive dependency.
    # - pass: we checked proton-bridge's Linux keychain selection code
    #   (pkg/keychain/helper_linux.go): the runtime backends are pass-app,
    #   secret-service, and secret-service-dbus. For a standalone container,
    #   Secret Service is the only alternative to pass, but it needs a working
    #   DBus/secret-service stack and is more operationally complex than a
    #   self-contained pass+GPG setup. Bridge uses pass as a real runtime
    #   keychain backend, not just during bootstrap.
    # - socat: Bridge listens on 127.0.0.1 internally, so we keep socat as the
    #   compatibility shim that exposes IMAP and SMTP outside the container.
    && apt-get install -y --no-install-recommends bash ca-certificates gnupg libfido2-1 libsecret-1-0 pass socat \
    && rm -rf /var/lib/apt/lists/*

RUN install -d -m 755 /protonmail

COPY --chmod=644 gpgparams /protonmail/gpgparams
COPY --chmod=755 entrypoint.sh /protonmail/entrypoint.sh
COPY --from=build /build/proton-bridge/bridge /protonmail/bridge

ENV XDG_CONFIG_HOME=/data/config \
    XDG_DATA_HOME=/data/data \
    XDG_CACHE_HOME=/data/cache \
    GNUPGHOME=/data/gpg

RUN useradd --create-home --home-dir /home/protonmail --shell /bin/bash protonmail \
    && install -d -o protonmail -g protonmail /data \
    && install -d -m 700 -o protonmail -g protonmail \
        /data/cache \
        /data/config \
        /data/data \
        /data/pass \
        /data/gpg

USER protonmail
VOLUME /data
WORKDIR /home/protonmail

ENTRYPOINT ["/protonmail/entrypoint.sh"]
CMD ["run"]
