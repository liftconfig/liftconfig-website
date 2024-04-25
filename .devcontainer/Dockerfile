FROM mcr.microsoft.com/devcontainers/base:jammy

RUN apt-get update && apt-get install -y \
    wget \
    unzip \
    python3 \
    python3-pip \
    git \
    libcairo2-dev libfreetype6-dev libffi-dev libjpeg-dev libpng-dev libz-dev

RUN pip install \
    mkdocs-material==9.5.18 \
    mkdocs-material[imaging] \
    mkdocs-rss-plugin==1.12.1 \
    pytz