FROM mcr.microsoft.com/devcontainers/base:ubuntu

# [Optional] Install Python, C++, Rust, and other build tools
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        python3 python3-pip python3-venv \
        build-essential \
        clang \
        curl \
        git \
        sudo \
        ca-certificates \
        pkg-config \
        rustc cargo \
    && rm -rf /var/lib/apt/lists/*

# Create a non-root user (vscode)
ARG USERNAME=vscode
ARG USER_UID=1000
ARG USER_GID=1000
RUN groupadd --gid $USER_GID $USERNAME \
    && useradd --uid $USER_UID --gid $USER_GID -m $USERNAME \
    && echo "$USERNAME ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers.d/$USERNAME \
    && chmod 0440 /etc/sudoers.d/$USERNAME

USER $USERNAME
WORKDIR /workspace
