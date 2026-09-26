# ==========================================================
# Stage 1 - Java 17 toolchain
# ==========================================================

FROM eclipse-temurin:17-jdk AS java17


# ==========================================================
# Stage 2 - Jenkins Agent
# ==========================================================

FROM jenkins/inbound-agent:latest-jdk21

USER root

ENV DEBIAN_FRONTEND=noninteractive

# ----------------------------------------------------------
# Copy Java 17 from official Temurin image
# ----------------------------------------------------------

COPY --from=java17 /opt/java/openjdk /opt/java/openjdk17

ENV JAVA17_HOME=/opt/java/openjdk17


# ----------------------------------------------------------
# Base utilities
# ----------------------------------------------------------

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        git \
        curl \
        wget \
        jq \
        unzip \
        ca-certificates \
        openssh-client \
        rsync \
        tar \
        bash \
        python3 \
        make \
        gnupg \
    && rm -rf /var/lib/apt/lists/*


# ----------------------------------------------------------
# Node.js 22
# ----------------------------------------------------------

RUN curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
    && apt-get update \
    && apt-get install -y --no-install-recommends nodejs \
    && node --version \
    && npm --version \
    && rm -rf /var/lib/apt/lists/*


# ----------------------------------------------------------
# Google Cloud CLI
# ----------------------------------------------------------

RUN mkdir -p /usr/share/keyrings \
    && curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg \
       | gpg --dearmor \
       -o /usr/share/keyrings/cloud.google.gpg \
    && echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" \
       > /etc/apt/sources.list.d/google-cloud-sdk.list \
    && apt-get update \
    && CLOUDSDK_SKIP_PY_COMPILATION=1 \
       apt-get install -y --no-install-recommends \
       google-cloud-cli \
    && gcloud --version \
    && rm -rf /var/lib/apt/lists/*


# ----------------------------------------------------------
# Jenkins Agent
# ----------------------------------------------------------

USER jenkins

WORKDIR /home/jenkins/agent