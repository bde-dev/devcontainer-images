ARG BASE=harbor.bde-dev.net/devcontainer-images/base
FROM ${BASE}

COPY --chown=vscode:vscode dotnet/mise.toml /mise/conf.d/20-dotnet.toml
COPY --chown=vscode:vscode dotnet/mise.lock /mise/dotnet.lock.tmp

RUN cat /mise/dotnet.lock.tmp >> /mise/mise.lock \
 && rm /mise/dotnet.lock.tmp \
 && mise install
