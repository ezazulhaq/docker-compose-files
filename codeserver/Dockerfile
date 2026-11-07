FROM codercom/code-server:4.140.0

LABEL org.opencontainers.image.source=https://github.com/ezazulhaq/docker-compose-files

USER root

# Install Python 3, JDK 21, Maven, Docker CLI, and necessary tools
RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y \
    python3 \
    python3-pip \
    python3-venv \
    openjdk-21-jdk \
    maven \
    curl \
    git \
    wget \
    gnupg \
    && mkdir -p -m 755 /etc/apt/keyrings \
    && wget -qO- https://cli.github.com/packages/githubcli-archive-keyring.gpg | tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null \
    && chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
    && curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc \
    && chmod a+r /etc/apt/keyrings/docker.asc \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null \
    && apt-get update \
    && apt-get install -y --no-install-recommends gh docker-ce-cli docker-compose-plugin docker-buildx-plugin \
    && rm -rf /var/lib/apt/lists/*

ENV JAVA_HOME=/usr/lib/jvm/java-21-openjdk-arm64
ENV PATH=$JAVA_HOME/bin:$PATH
ENV DOCKER_CONFIG=/home/coder/.docker

# Change coder user uid/gid to 1001 to match host workspace permissions, setup docker group and config directory
RUN usermod -u 1001 coder && groupmod -g 1001 coder \
    && (groupadd -g 988 docker 2>/dev/null || groupadd -f docker) \
    && usermod -aG docker coder \
    && mkdir -p /home/coder/.docker /mnt/data/docker \
    && chown -R 1001:1001 /home/coder

# Switch back to the coder user (now UID 1001)
USER 1001

# Install nvm (Node Version Manager) and Node.js version 24
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash \
    && export NVM_DIR="$HOME/.nvm" \
    && [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh" \
    && nvm install 24 \
    && nvm alias default 24

# Install uv (fast Python package installer)
RUN curl -LsSf https://astral.sh/uv/install.sh | sh

# Install Antigravity CLI
RUN curl -fsSL https://antigravity.google/cli/install.sh | bash

# Pre-install VS Code extensions
RUN extensions="angular.ng-template \
    dbaeumer.vscode-eslint \
    eamodio.gitlens \
    esbenp.prettier-vscode \
    pkief.material-icon-theme \
    humao.rest-client \
    jasonnutter.search-node-modules \
    mgmcdermott.vscode-language-babel \
    ms-azuretools.vscode-docker \
    redhat.java \
    sonarsource.sonarlint-vscode \
    vmware.vscode-boot-dev-pack \
    vmware.vscode-spring-boot \
    vscjava.vscode-gradle \
    vscjava.vscode-java-debug \
    vscjava.vscode-java-dependency \
    vscjava.vscode-java-pack \
    vscjava.vscode-java-test \
    vscjava.vscode-maven \
    vscjava.vscode-spring-boot-dashboard \
    vscjava.vscode-spring-initializr \
    ms-python.python \
    ms-python.debugpy \
    ms-python.vscode-pylance \
    ms-python.vscode-python-envs \
    njpwerner.autodocstring" && \
    for ext in $extensions; do \
        code-server --install-extension $ext || echo "Warning: Failed to install $ext, it might not be available in Open VSX."; \
    done
