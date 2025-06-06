FROM debian:bookworm-slim

# Set noninteractive frontend for apt-get
ENV DEBIAN_FRONTEND=noninteractive

# Install base dependencies, docker prerequisites, Go, PAM dev files, user tools
RUN apt-get update && apt-get install -y --no-install-recommends \
    # Docker prerequisites & tools
    ca-certificates \
    curl \
    wget \
    gnupg \
    lsb-release \
    iptables \
    uidmap \
    # API Server build/runtime deps
    build-essential \
    libpam0g-dev \
    login \
    git \
    # Containerlab deps & general utils
    bash \
    iproute2 \
    sudo \
    procps \
    openssh-client \
 && rm -rf /var/lib/apt/lists/*

# --- Install Docker Engine (following official docs) ---
# Add Docker's official GPG key
RUN install -m 0755 -d /etc/apt/keyrings
RUN curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
RUN chmod a+r /etc/apt/keyrings/docker.asc

# Add the Docker repository
RUN echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine, CLI, Containerd, Buildx, Compose
RUN apt-get update && apt-get install -y --no-install-recommends \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    docker-buildx-plugin \
    docker-compose-plugin \
 && rm -rf /var/lib/apt/lists/*

# Verify Docker installation
RUN docker --version

# --- Install Go (architecture-aware) ---
ARG GO_VERSION=1.22.2
ENV PATH="/usr/local/go/bin:${PATH}"

# Detect architecture and download appropriate Go binary
RUN ARCH=$(dpkg --print-architecture) && \
    case ${ARCH} in \
      amd64) GO_ARCH="amd64" ;; \
      arm64) GO_ARCH="arm64" ;; \
      *) echo "Unsupported architecture: ${ARCH}"; exit 1 ;; \
    esac && \
    curl -fsSL "https://golang.org/dl/go${GO_VERSION}.linux-${GO_ARCH}.tar.gz" | tar -C /usr/local -xz

# Verify Go installation
RUN go version

# --- Install Containerlab ---
# Use ARG for flexibility, default to latest if not provided during build
ARG CLAB_VERSION=""
RUN if [ -z "$CLAB_VERSION" ]; then \
        echo "Installing latest Containerlab..."; \
        bash -c "$(curl -sL https://get.containerlab.dev)"; \
    else \
        echo "Installing Containerlab version ${CLAB_VERSION}..."; \
        bash -c "$(curl -sL https://get.containerlab.dev)" -- -v ${CLAB_VERSION}; \
    fi
# Verify Containerlab installation
RUN containerlab version

ARG API_USER=admin
ARG API_PASS=admin
RUN useradd -m -s /bin/bash ${API_USER} \
 && (getent group clab_admins || groupadd clab_admins) \
 && usermod -aG clab_admins,docker ${API_USER} \
 && usermod -aG clab_admins,docker root \
 && echo "${API_USER}:${API_PASS}" | chpasswd \
 && echo "INFO: Created user '${API_USER}' with provided password."

WORKDIR /app

# Copy Go module files first for caching
COPY go.mod go.sum ./
RUN go mod download

# Copy the rest of the application source
COPY . .

# Build the clab-api-server binary with CGO enabled for PAM
RUN CGO_ENABLED=1 GOOS=linux go build -v -ldflags="\
    -X main.version=debian-dind-$(cat VERSION 2>/dev/null || echo dev) \
    -X main.commit=$(git rev-parse --short HEAD 2>/dev/null || echo unknown) \
    -X main.date=$(date -u +'%Y-%m-%dT%H:%M:%SZ')" \
    -o /usr/local/bin/clab-api-server ./cmd/server

# --- Configure API Server ---
# Copy templates
RUN mkdir -p /templates && cp -r templates/redoc.html /templates/

# --- DIND Setup ---
COPY docker/common/user-persistence.sh /usr/local/lib/user-persistence.sh
COPY docker/dind/entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/lib/user-persistence.sh /usr/local/bin/entrypoint.sh

# Define volumes for persistence (inner docker data and user homes)
VOLUME /var/lib/docker
VOLUME /home
VOLUME /persistent-config

# Use ARGs for ports, falling back to defaults if not provided
ARG API_PORT=8080
ARG SSH_BASE_PORT=2223
ARG SSH_MAX_PORT=2322
EXPOSE ${API_PORT}
EXPOSE ${SSH_BASE_PORT}-${SSH_MAX_PORT}
EXPOSE 22

# Set the ENTRYPOINT to our custom script
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]

# Default command passed to our entrypoint script
CMD ["clab-api-server"]