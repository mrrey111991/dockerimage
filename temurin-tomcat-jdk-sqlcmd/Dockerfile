# ==============================================================================
# Base Image
# Eclipse Temurin Java 21 - UBI 9 Minimal
# ==============================================================================

FROM eclipse-temurin:21-jdk-ubi9-minimal

ENV ACCEPT_EULA=Y
ENV CATALINA_HOME=/usr/local/tomcat
ENV PATH="${CATALINA_HOME}/bin:/opt/mssql-tools18/bin:${PATH}"

# ==============================================================================
# Arguments
# ==============================================================================

ARG TOMCAT_VERSION=11.0.24
ARG TOMCAT_BASE=apache-tomcat-${TOMCAT_VERSION}

ARG TOMCAT_URL=https://downloads.apache.org/tomcat/tomcat-11/v${TOMCAT_VERSION}/bin/${TOMCAT_BASE}.tar.gz
ARG TOMCAT_ARCHIVE=https://archive.apache.org/dist/tomcat/tomcat-11/v${TOMCAT_VERSION}/bin/${TOMCAT_BASE}.tar.gz

ARG TOMCAT_SHA512=""

# ==============================================================================
# Root
# ==============================================================================

USER root

# ==============================================================================
# Update Base Image
# ==============================================================================

RUN set -eux; \
    microdnf update -y; \
    microdnf clean all; \
    rm -rf /var/cache/yum /var/cache/dnf

# ==============================================================================
# Install Required Packages
#
# Required by application:
#   ca-certificates
#   file
#   tar
#   gzip
#   unzip
#   openssl
#   libpng
#   zlib
#
# Do NOT install full curl because UBI minimal already provides curl-minimal.
# ==============================================================================

RUN set -eux; \
    microdnf install -y \
        ca-certificates \
        file \
        tar \
        gzip \
        unzip \
        openssl \
        libpng \
        zlib; \
    microdnf update -y; \
    microdnf clean all; \
    rm -rf /var/cache/yum /var/cache/dnf

# ==============================================================================
# Install Microsoft SQLCMD
# Microsoft RHEL 9 Repository
# ==============================================================================

RUN set -eux; \
    mkdir -p /etc/yum.repos.d; \
    curl -fsSL \
        https://packages.microsoft.com/config/rhel/9/prod.repo \
        -o /etc/yum.repos.d/microsoft-prod.repo; \
    \
    microdnf update -y; \
    \
    ACCEPT_EULA=Y microdnf install -y \
        mssql-tools18 \
        unixODBC; \
    \
    ln -sf \
        /opt/mssql-tools18/bin/sqlcmd \
        /usr/local/bin/sqlcmd; \
    \
    microdnf update -y; \
    microdnf clean all; \
    rm -rf /var/cache/yum /var/cache/dnf

# ==============================================================================
# Create Tomcat User
# ==============================================================================

RUN set -eux; \
    groupadd --system tomcat; \
    useradd \
        --system \
        --gid tomcat \
        --home-dir "${CATALINA_HOME}" \
        --shell /sbin/nologin \
        tomcat

# ==============================================================================
# Install Apache Tomcat
# ==============================================================================

RUN set -eux; \
    mkdir -p "${CATALINA_HOME}"; \
    \
    if curl -fsSL --retry 5 \
        -o /tmp/tomcat.tar.gz \
        "${TOMCAT_URL}"; then \
        echo "Downloaded Tomcat from primary mirror"; \
    else \
        echo "Primary mirror failed, using Apache archive"; \
        curl -fsSL --retry 5 \
            -o /tmp/tomcat.tar.gz \
            "${TOMCAT_ARCHIVE}"; \
    fi; \
    \
    if curl -fsSL \
        -o /tmp/tomcat.sha512 \
        "${TOMCAT_URL}.sha512"; then \
        awk '{print $1 "  /tmp/tomcat.tar.gz"}' \
            /tmp/tomcat.sha512 \
            > /tmp/check.sha512; \
        sha512sum -c /tmp/check.sha512; \
        rm -f /tmp/check.sha512 /tmp/tomcat.sha512; \
    elif [ -n "${TOMCAT_SHA512}" ]; then \
        echo "${TOMCAT_SHA512}  /tmp/tomcat.tar.gz" \
            > /tmp/check.sha512; \
        sha512sum -c /tmp/check.sha512; \
        rm -f /tmp/check.sha512; \
    else \
        echo "WARNING: Tomcat SHA512 verification skipped."; \
    fi; \
    \
    tar -xzf /tmp/tomcat.tar.gz \
        --strip-components=1 \
        -C "${CATALINA_HOME}"; \
    \
    rm -f /tmp/tomcat.tar.gz; \
    chmod +x "${CATALINA_HOME}"/bin/*.sh; \
    chown -R tomcat:tomcat "${CATALINA_HOME}"

# ==============================================================================
# Final Security Update
# ==============================================================================

RUN set -eux; \
    microdnf update -y; \
    microdnf clean all; \
    rm -rf /var/cache/yum /var/cache/dnf

# ==============================================================================
# Verify Required Packages
# ==============================================================================

RUN set -eux; \
    echo "========================================"; \
    echo "Required Package Versions"; \
    echo "========================================"; \
    rpm -q ca-certificates; \
    rpm -q file; \
    rpm -q tar; \
    rpm -q gzip; \
    rpm -q unzip; \
    rpm -q openssl; \
    rpm -q libpng; \
    rpm -q zlib; \
    \
    echo "========================================"; \
    echo "System Libraries"; \
    echo "========================================"; \
    rpm -q glibc; \
    \
    echo "========================================"; \
    echo "Java"; \
    echo "========================================"; \
    java -version; \
    \
    echo "========================================"; \
    echo "OpenSSL"; \
    echo "========================================"; \
    openssl version; \
    \
    echo "========================================"; \
    echo "SQLCMD"; \
    echo "========================================"; \
    sqlcmd -? || true; \
    \
    echo "========================================"; \
    echo "Tomcat"; \
    echo "========================================"; \
    "${CATALINA_HOME}/bin/version.sh"

# ==============================================================================
# Runtime
# ==============================================================================

EXPOSE 8080

USER tomcat

WORKDIR ${CATALINA_HOME}

CMD ["catalina.sh", "run"]