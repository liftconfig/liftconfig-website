FROM ubuntu:24.04

RUN apt-get update && apt-get install -y \
    wget \
    unzip \
    python3 \
    python3-venv \
    python3-pip \
    git \
    libcairo2-dev libfreetype6-dev libffi-dev libjpeg-dev libpng-dev libz-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /usr/src/mkdocs-website

RUN python3 -m venv /usr/src/venv

ENV PATH="/usr/src/venv/bin:$PATH"

RUN pip install --upgrade pip \
    && pip install \
    mkdocs-material==9.5.49 \
    mkdocs-material[imaging] \
    mkdocs-rss-plugin==1.17.1 \
    pytz
