ARG BASE=harbor.bde-dev.net/devcontainers/base
FROM ${BASE}

COPY dotnet/mise.toml /etc/mise/conf.d/20-dotnet.toml

RUN /usr/local/bin/mise install
