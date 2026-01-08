# Base image
FROM ubuntu:22.04


# Install dependencies
RUN apt update && apt install -y \
    build-essential \
    qemu-system \
    gdb \
    make \
    git \
    sudo \
    vim \
    && apt clean

COPY ../pintos/src/utils/pintos /usr/local/bin/pintos
RUN chmod +x /usr/local/bin/pintos

COPY ../pintos/src/utils/Pintos.pm /usr/local/bin/Pintos.pm
RUN chmod +x /usr/local/bin/Pintos.pm

# Set up a user for development
ARG USERNAME=vscode
ARG USER_UID=1000
ARG USER_GID=$USER_UID
RUN groupadd --gid $USER_GID $USERNAME \
    && useradd --uid $USER_UID --gid $USER_GID -m $USERNAME \
    && echo "$USERNAME ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME \
    && chmod 0440 /etc/sudoers.d/$USERNAME

# Set the default user
USER $USERNAME