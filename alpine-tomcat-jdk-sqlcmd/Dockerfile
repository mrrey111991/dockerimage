# ============================================================
# Stage 1: Build SQLCMD
# ============================================================

FROM golang:1.26.5-alpine AS sqlcmd-builder

WORKDIR /build

# ============================================================
# Build dependencies
# ============================================================

RUN set -eux; \
    apk add --no-cache \
        git \
        ca-certificates

# ============================================================
# Download CURRENT SQLCMD source
#
# We intentionally use the current source instead of the
# v1.10.0 tag because v1.10.0 contains the old:
#
#   github.com/docker/docker 28.5.2+incompatible
#
# Current SQLCMD has migrated to Moby modules.
# ============================================================

RUN set -eux; \
    git clone --depth 1 \
        "https://github.com/microsoft/go-sqlcmd.git" \
        /build/go-sqlcmd

WORKDIR /build/go-sqlcmd

# ============================================================
# Upgrade Go security dependencies
# ============================================================

RUN set -eux; \
    go mod edit \
        -require=golang.org/x/crypto@v0.54.0 \
        -require=golang.org/x/net@v0.57.0; \
    go mod tidy; \
    go mod download

# ============================================================
# Verify vulnerable Docker dependency is NOT present
# ============================================================

RUN set -eux; \
    echo "Checking Go dependencies..."; \
    if go list -m all | grep -q '^github.com/docker/docker '; then \
        echo "ERROR: github.com/docker/docker dependency detected"; \
        go list -m all | grep '^github.com/docker/docker '; \
        exit 1; \
    fi; \
    echo "OK: github.com/docker/docker is NOT present"

# ============================================================
# Display security-related dependency versions
# ============================================================

RUN set -eux; \
    echo "========================================"; \
    echo "Go crypto:"; \
    go list -m golang.org/x/crypto; \
    echo "Go network:"; \
    go list -m golang.org/x/net; \
    echo "========================================"

# ============================================================
# Build SQLCMD
# ============================================================

RUN set -eux; \
    CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \
    go build \
        -trimpath \
        -ldflags="-s -w" \
        -o /sqlcmd \
        ./cmd/modern

# ============================================================
# Verify SQLCMD
# ============================================================

RUN set -eux; \
    /sqlcmd --version


# ============================================================
# Stage 2: Runtime Image
# ============================================================

FROM alpine:3.24

ARG TOMCAT_VERSION=11.0.24

# ============================================================
# Environment
# ============================================================

ENV JAVA_HOME=/usr/lib/jvm/java-21-openjdk \
    CATALINA_HOME=/usr/local/tomcat \
    CATALINA_BASE=/usr/local/tomcat \
    PATH=/usr/local/tomcat/bin:/usr/local/bin:$PATH \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8

# ============================================================
# Runtime dependencies
#
# curl  = HTTP requests / downloads
# unzip = ZIP extraction
# ============================================================

RUN set -eux; \
    apk update; \
    apk upgrade; \
    apk add --no-cache \
        openjdk21-jre-headless \
        ca-certificates \
        bash \
        tzdata \
        openssl \
        curl \
        unzip; \
    update-ca-certificates; \
    java -version; \
    curl --version; \
    unzip -v | head -n 3

# ============================================================
# Install Apache Tomcat 11
# ============================================================

RUN set -eux; \
    mkdir -p "${CATALINA_HOME}"; \
    curl -fsSL --retry 5 \
        -o /tmp/tomcat.tar.gz \
        "https://dlcdn.apache.org/tomcat/tomcat-11/v${TOMCAT_VERSION}/bin/apache-tomcat-${TOMCAT_VERSION}.tar.gz"; \
    tar -xzf /tmp/tomcat.tar.gz \
        --strip-components=1 \
        -C "${CATALINA_HOME}"; \
    rm -f /tmp/tomcat.tar.gz; \
    rm -rf "${CATALINA_HOME}/webapps/"*; \
    mkdir -p "${CATALINA_HOME}/webapps"; \
    chmod +x "${CATALINA_HOME}/bin/"*.sh; \
    "${CATALINA_HOME}/bin/catalina.sh" version

# ============================================================
# Install SQLCMD
# ============================================================

COPY --from=sqlcmd-builder \
    /sqlcmd \
    /usr/local/bin/sqlcmd

RUN set -eux; \
    chmod 0755 /usr/local/bin/sqlcmd; \
    /usr/local/bin/sqlcmd --version

# ============================================================
# Create Tomcat user
# ============================================================

RUN set -eux; \
    addgroup -S tomcat; \
    adduser -S -D -H \
        -s /sbin/nologin \
        -G tomcat \
        tomcat; \
    chown -R tomcat:tomcat "${CATALINA_HOME}"; \
    chmod +x "${CATALINA_HOME}/bin/"*.sh

# ============================================================
# Working directory
# ============================================================

WORKDIR ${CATALINA_HOME}

# ============================================================
# Expose Tomcat
# ============================================================

EXPOSE 8080

# ============================================================
# Run as non-root
# ============================================================

USER tomcat

# ============================================================
# Start Tomcat
# ============================================================

CMD ["catalina.sh", "run"]