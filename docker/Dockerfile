FROM python:3.11-slim-bullseye as sdist

LABEL maintainer="oss@netflix.com"
LABEL org.opencontainers.image.title="Dispatch PyPI Wheel"
LABEL org.opencontainers.image.description="PyPI Wheel Builder for Dispatch"
LABEL org.opencontainers.image.url="https://dispatch.io/"
LABEL org.opencontainers.image.source="https://github.com/netflix/dispatch"
LABEL org.opencontainers.image.vendor="Netflix, Inc."
LABEL org.opencontainers.image.authors="oss@netflix.com"

SHELL ["/bin/bash", "-o", "pipefail", "-o", "errexit", "-c"]

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    gcc \
    g++ \
    gfortran \
    libatlas-base-dev \
    libopenblas-dev \
    pkg-config \
    wget \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Set up Node.js
RUN wget --quiet -O - https://deb.nodesource.com/setup_16.x | bash - \
    && apt-get install -y nodejs --no-install-recommends

# Set environment variables
ENV MAKEFLAGS="-j$(nproc)"
ENV DISPATCH_CONF=/etc/dispatch

# Upgrade pip and install wheel
RUN pip install --no-cache-dir --upgrade pip setuptools wheel

ARG SOURCE_COMMIT
ENV DISPATCH_BUILD=${SOURCE_COMMIT:-unknown}
LABEL org.opencontainers.image.revision=$SOURCE_COMMIT
LABEL org.opencontainers.image.licenses="https://github.com/netflix/dispatch/blob/${SOURCE_COMMIT:-master}/LICENSE"

ARG DISPATCH_LIGHT_BUILD
ENV DISPATCH_LIGHT_BUILD=${DISPATCH_LIGHT_BUILD}

RUN echo "DISPATCH_LIGHT_BUILD=${DISPATCH_LIGHT_BUILD}"

# Allow build time variables via --build-arg
ARG VITE_DISPATCH_AUTH_REGISTRATION_ENABLED
ARG VITE_DISPATCH_AUTHENTICATION_PROVIDER_PKCE_CLIENT_ID
ARG VITE_DISPATCH_AUTHENTICATION_PROVIDER_PKCE_OPEN_ID_CONNECT_URL
ARG VITE_DISPATCH_AUTHENTICATION_PROVIDER_SLUG
ARG VITE_DISPATCH_AUTHENTICATION_PROVIDER_USE_ID_TOKEN
ARG VITE_SENTRY_DSN
ARG VITE_SENTRY_APP_KEY
ARG VITE_SENTRY_ENABLED

# Should be replaced in your build process script
ARG VITE_DISPATCH_COMMIT_HASH
ENV VITE_DISPATCH_COMMIT_HASH="Unknown"

ARG VITE_DISPATCH_COMMIT_MESSAGE
ENV VITE_DISPATCH_COMMIT_MESSAGE="Unknown"

COPY . /usr/src/dispatch/
RUN YARN_CACHE_FOLDER="$(mktemp -d)" \
    && export YARN_CACHE_FOLDER \
    && pushd /usr/src/dispatch \
    && pip install --no-cache-dir -r requirements.txt \
    && python setup.py bdist_wheel \
    && rm -r "$YARN_CACHE_FOLDER" \
    && mv /usr/src/dispatch/dist /dist

# This is the image to be run
FROM python:3.11-slim-bullseye

LABEL maintainer="oss@dispatch.io"
LABEL org.opencontainers.image.title="Dispatch"
LABEL org.opencontainers.image.description="Dispatch runtime image"
LABEL org.opencontainers.image.url="https://github.com/netflix/dispatch"
LABEL org.opencontainers.image.documentation="https://github.com/netflix/dispatch"
LABEL org.opencontainers.image.source="https://github.com/netflix/dispatch"
LABEL org.opencontainers.image.vendor="Netflix, Inc."
LABEL org.opencontainers.image.authors="oss@netflix.com"

SHELL ["/bin/bash", "-o", "pipefail", "-o", "errexit", "-c"]

# add our user and group first to make sure their IDs get assigned consistently
RUN groupadd -r dispatch && useradd -r -m -g dispatch dispatch

# Install runtime dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    nodejs \
    && rm -rf /var/lib/apt/lists/*

# Copy the built wheel from the sdist stage
COPY --from=sdist /dist/*.whl /tmp/dist/

# Install the wheel
RUN pip install --no-cache-dir /tmp/dist/*.whl \
    && rm -rf /tmp/dist

# Install mjml
RUN npm install -g mjml --no-cache

EXPOSE 8000
VOLUME /var/lib/dispatch/files

ENTRYPOINT ["dispatch"]
CMD ["server", "start", "dispatch.main:app", "--host=0.0.0.0"]

ARG SOURCE_COMMIT
LABEL org.opencontainers.image.revision=$SOURCE_COMMIT
LABEL org.opencontainers.image.licenses="https://github.com/netflix/dispatch/blob/${SOURCE_COMMIT:-master}/LICENSE"
