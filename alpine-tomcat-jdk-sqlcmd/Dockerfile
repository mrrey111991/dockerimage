# ===== Stage 0: Build sqlcmd =====
FROM golang:1.25.5-alpine3.22 AS build

# Install dependencies for building sqlcmd
RUN apk add --no-cache git bash file zlib gnupg libtasn1 libpng libc6-compat curl

WORKDIR /src

# Clone and build sqlcmd
RUN git clone https://github.com/microsoft/go-sqlcmd.git . \
    && go build -o /go/bin/sqlcmd ./cmd/sqlcmd


# ===== Stage 1: Runtime image with Tomcat + sqlcmd =====
FROM eclipse-temurin:21-jdk-alpine

# Environment variables
ENV CATALINA_HOME=/usr/local/tomcat
ENV PATH="$CATALINA_HOME/bin:$PATH"

# Install runtime dependencies
RUN apk add --no-cache \
    bash \
    curl \
    tar \
    ca-certificates \
    libc6-compat \
    file \
    zlib \
    gnupg \
    libtasn1 \
    libpng \
    && update-ca-certificates \
    && rm -rf /var/cache/apk/*

# Copy sqlcmd from build stage
COPY --from=build /go/bin/sqlcmd /usr/local/bin/sqlcmd

# Download and extract Tomcat
ARG TOMCAT_VERSION=10.1.50
RUN mkdir -p "$CATALINA_HOME" \
    && base="apache-tomcat-${TOMCAT_VERSION}" \
    && urls="https://downloads.apache.org/tomcat/tomcat-10/v${TOMCAT_VERSION}/bin/${base}.tar.gz https://archive.apache.org/dist/tomcat/tomcat-10/v${TOMCAT_VERSION}/bin/${base}.tar.gz" \
    && for url in $urls; do \
        echo "Downloading: $url"; \
        if curl -fL --retry 5 --retry-delay 2 -o /tmp/tomcat.tgz "$url"; then break; fi; \
    done \
    && [ -s /tmp/tomcat.tgz ] \
    && tar -xzf /tmp/tomcat.tgz -C "$CATALINA_HOME" --strip-components=1 \
    && rm /tmp/tomcat.tgz \
    && chmod +x "$CATALINA_HOME"/bin/*.sh

# Expose Tomcat default port
EXPOSE 8080

# Default command
CMD ["catalina.sh", "run"]
