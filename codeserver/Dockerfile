FROM codercom/code-server:4.121.0

LABEL org.opencontainers.image.source=https://github.com/ezazulhaq/docker-compose-files

# Pre-install VS Code extensions
RUN extensions="angular.ng-template \
    dbaeumer.vscode-eslint \
    eamodio.gitlens \
    esbenp.prettier-vscode \
    google.google-antigravity \
    humao.rest-client \
    jasonnutter.search-node-modules \
    mgmcdermott.vscode-language-babel \
    ms-azuretools.vscode-containers \
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
        code-server --install-extension $ext \
        || echo "Warning: Failed to install $ext, it might not be available in Open VSX."; \
    done
