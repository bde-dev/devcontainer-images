# The base version this flavour stands on. A release builds on exactly this; the local
# ladder overrides it with the base built from the working tree.
ARG BASE=harbor.bde-dev.net/devcontainer-images/base:1.0.2
FROM ${BASE}

COPY --chown=vscode:vscode dotnet/mise.toml /mise/conf.d/20-dotnet.toml
COPY --chown=vscode:vscode dotnet/mise.lock /mise/dotnet.lock.tmp

RUN cat /mise/dotnet.lock.tmp >> /mise/mise.lock \
 && rm /mise/dotnet.lock.tmp \
 && mise install --jobs 1
