FROM mcr.microsoft.com/vscode/devcontainers/ruby:0-2

# ENV Variables required by Jekyll
ENV LANG=da_DK.UTF-8 \
    LANGUAGE=da_DK:en \
    TZ=Europe/Copenhagen \
    LC_ALL=da_DK.UTF-8 \
    LANG=da_DK.UTF-8 \
    LANGUAGE=da_DK

# Install bundler
RUN gem install bundler

# [Option] Install Node.js
ARG NODE_VERSION="lts/*"
RUN su vscode -c "source /usr/local/share/nvm/nvm.sh && nvm install ${NODE_VERSION} 2>&1"

# [Optional] Uncomment this section to install additional OS packages.
# RUN apt-get update && export DEBIAN_FRONTEND=noninteractive \
#     && apt-get -y install --no-install-recommends <your-package-list-here>

# [Optional] Uncomment this line to install additional gems.
# RUN gem install <your-gem-names-here>

# [Optional] Uncomment this line to install global node packages.
# RUN su vscode -c "source /usr/local/share/nvm/nvm.sh && npm install -g <your-package-here>" 2>&1

RUN echo "da_DK.UTF-8 UTF-8" > /etc/locale.gen && locale-gen

COPY library-scripts/github-debian.sh /tmp/library-scripts/
RUN apt-get update \
    && bash /tmp/library-scripts/github-debian.sh \
    && rm -rf /var/lib/apt/lists/*
