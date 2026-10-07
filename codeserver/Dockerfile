FROM codercom/code-server:4.121.0

LABEL org.opencontainers.image.source=https://github.com/ezazulhaq/docker-compose-files

USER root

# Install Python 3, JDK 21, Maven, and necessary tools
RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y \
    python3 \
    python3-pip \
    python3-venv \
    openjdk-21-jdk \
    maven \
    curl \
    git \
    && rm -rf /var/lib/apt/lists/*

ENV JAVA_HOME=/usr/lib/jvm/java-21-openjdk-arm64
ENV PATH=$JAVA_HOME/bin:$PATH

# Switch back to the default coder user
USER 1000

# Install nvm (Node Version Manager) and Node.js version 24
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash \
    && export NVM_DIR="$HOME/.nvm" \
    && [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh" \
    && nvm install 24 \
    && nvm alias default 24

# Install uv (fast Python package installer)
RUN curl -LsSf https://astral.sh/uv/install.sh | sh

# Pre-install VS Code extensions
RUN extensions="angular.ng-template \
    dbaeumer.vscode-eslint \
    eamodio.gitlens \
    esbenp.prettier-vscode \
    google.google-antigravity \
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
    vscjava.vscode-spring-initializr" && \
    for ext in $extensions; do \
        code-server --install-extension $ext || echo "Warning: Failed to install $ext, it might not be available in Open VSX."; \
    done
