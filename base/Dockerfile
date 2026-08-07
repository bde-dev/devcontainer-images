FROM mcr.microsoft.com/devcontainers/base:ubuntu-24.04

COPY --from=ghcr.io/jdx/mise:2026.8.2 /usr/local/bin/mise /usr/local/bin/

COPY base/mise.toml /etc/mise/conf.d/10-base.toml

USER vscode

RUN /usr/local/bin/mise install

RUN echo 'eval "$(mise activate bash)"' >> /home/vscode/.bashrc && \
  echo 'eval "$(mise activate zsh)"' >> /home/vscode/.zshrc
