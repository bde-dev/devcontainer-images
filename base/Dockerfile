FROM mcr.microsoft.com/devcontainers/base:ubuntu-24.04

COPY --from=ghcr.io/jdx/mise:2026.10.23 /usr/local/bin/mise /usr/local/bin/

ENV MISE_DATA_DIR=/mise \
    MISE_CONFIG_DIR=/mise \
    MISE_CACHE_DIR=/mise/cache \
    MISE_TRUSTED_CONFIG_PATHS=/mise \
    PATH=/mise/shims:$PATH

COPY base/mise.toml /mise/config.toml
COPY base/mise.lock /mise/mise.lock

RUN chown -R vscode:vscode /mise

USER vscode

RUN mise install
