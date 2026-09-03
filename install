#!/usr/bin/env bash

set -Eeuo pipefail

# ============================================================
# SELF-HOSTED SUPABASE AUTO INSTALLER
# Ubuntu 24.04 + Docker + Caddy + Cloudflare
#
# Architecture:
#   web.ummahlifestyle.com    -> Lovable/frontend
#   api.ummahlifestyle.com    -> Supabase API
#   studio.ummahlifestyle.com -> Supabase Studio
#
# Public ports:
#   22, 80, 443
#
# Private:
#   PostgreSQL 5432
#   Supavisor 6543
#   Supabase internal services
#
# Run on a FRESH Ubuntu 24.04 server.
# ============================================================


# ============================================================
# CONFIGURATION
# ============================================================

DOMAIN="ummahlifestyle.com"
API_DOMAIN="api.${DOMAIN}"
STUDIO_DOMAIN="studio.${DOMAIN}"
WEB_DOMAIN="web.${DOMAIN}"

ADMIN_USER="admin"

SUPABASE_DIR="/opt/supabase"

# Leave empty to use the existing root SSH authorized key.
SSH_PUBLIC_KEY=""

# Set to "yes" if you want the script to install everything.
AUTO_REBOOT="no"


# ============================================================
# COLORS / LOGGING
# ============================================================

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

log() {
    echo -e "${GREEN}[+]${NC} $1"
}

warn() {
    echo -e "${YELLOW}[!]${NC} $1"
}

error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

die() {
    error "$1"
    exit 1
}


# ============================================================
# ERROR HANDLER
# ============================================================

trap 'error "Installation failed at line $LINENO. Check the command above."' ERR


# ============================================================
# ROOT CHECK
# ============================================================

if [[ "${EUID}" -ne 0 ]]; then
    die "Run this script as root: sudo bash install-supabase.sh"
fi


# ============================================================
# OS CHECK
# ============================================================

if [[ ! -f /etc/os-release ]]; then
    die "Cannot detect operating system."
fi

source /etc/os-release

if [[ "${ID}" != "ubuntu" ]]; then
    die "This installer is designed for Ubuntu."
fi

if [[ "${VERSION_ID}" != "24.04" ]]; then
    warn "This script was designed for Ubuntu 24.04."
    warn "Detected Ubuntu ${VERSION_ID}."
fi

log "Operating system: ${PRETTY_NAME}"


# ============================================================
# BASIC SERVER INFORMATION
# ============================================================

SERVER_IP=$(hostname -I | awk '{print $1}')

log "Server IP: ${SERVER_IP}"
log "Domain: ${DOMAIN}"
log "API: ${API_DOMAIN}"
log "Studio: ${STUDIO_DOMAIN}"
log "Web: ${WEB_DOMAIN}"

echo
echo "============================================================"
echo " SUPABASE INSTALLATION"
echo "============================================================"
echo
echo "This script will configure:"
echo
echo "  ${API_DOMAIN}"
echo "  ${STUDIO_DOMAIN}"
echo "  ${WEB_DOMAIN}"
echo
echo "Public ports: 22, 80, 443"
echo
read -r -p "Continue? [y/N]: " CONFIRM

if [[ ! "${CONFIRM}" =~ ^[Yy]$ ]]; then
    echo "Cancelled."
    exit 0
fi


# ============================================================
# SYSTEM UPDATE
# ============================================================

log "Updating Ubuntu..."

export DEBIAN_FRONTEND=noninteractive

apt-get update
apt-get upgrade -y


# ============================================================
# INSTALL BASE PACKAGES
# ============================================================

log "Installing required packages..."

apt-get install -y \
    ca-certificates \
    curl \
    wget \
    git \
    gnupg \
    lsb-release \
    ufw \
    fail2ban \
    unattended-upgrades \
    apt-transport-https \
    software-properties-common


# ============================================================
# CREATE ADMIN USER
# ============================================================

if id "${ADMIN_USER}" >/dev/null 2>&1; then
    log "User ${ADMIN_USER} already exists."
else
    log "Creating ${ADMIN_USER} user..."

    adduser --disabled-password --gecos "" "${ADMIN_USER}"
fi

usermod -aG sudo "${ADMIN_USER}"


# ============================================================
# SSH KEY SETUP
# ============================================================

log "Configuring SSH key for ${ADMIN_USER}..."

mkdir -p "/home/${ADMIN_USER}/.ssh"

if [[ -n "${SSH_PUBLIC_KEY}" ]]; then
    echo "${SSH_PUBLIC_KEY}" > "/home/${ADMIN_USER}/.ssh/authorized_keys"

elif [[ -f /root/.ssh/authorized_keys ]]; then
    cp /root/.ssh/authorized_keys \
        "/home/${ADMIN_USER}/.ssh/authorized_keys"

else
    warn "No root authorized_keys file found."
    warn "You MUST add your SSH public key manually before logging out."
fi

chown -R "${ADMIN_USER}:${ADMIN_USER}" "/home/${ADMIN_USER}/.ssh"
chmod 700 "/home/${ADMIN_USER}/.ssh"

if [[ -f "/home/${ADMIN_USER}/.ssh/authorized_keys" ]]; then
    chmod 600 "/home/${ADMIN_USER}/.ssh/authorized_keys"
fi


# ============================================================
# SSH HARDENING
# ============================================================

log "Hardening SSH..."

mkdir -p /etc/ssh/sshd_config.d

cat > /etc/ssh/sshd_config.d/99-supabase-hardening.conf <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
UsePAM yes
X11Forwarding no
AllowUsers admin
EOF

sshd -t

systemctl reload ssh


# ============================================================
# UFW FIREWALL
# ============================================================

log "Configuring UFW..."

ufw --force reset

ufw default deny incoming
ufw default allow outgoing

ufw allow 22/tcp comment 'SSH'
ufw allow 80/tcp comment 'HTTP'
ufw allow 443/tcp comment 'HTTPS'

ufw --force enable

ufw status verbose


# ============================================================
# FAIL2BAN
# ============================================================

log "Configuring Fail2ban..."

mkdir -p /etc/fail2ban

cat > /etc/fail2ban/jail.local <<'EOF'
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 5
backend = systemd

[sshd]
enabled = true
port = 22
maxretry = 5
EOF

systemctl enable fail2ban
systemctl restart fail2ban


# ============================================================
# AUTOMATIC SECURITY UPDATES
# ============================================================

log "Enabling unattended security updates..."

dpkg-reconfigure -f noninteractive unattended-upgrades || true

systemctl enable unattended-upgrades
systemctl restart unattended-upgrades || true


# ============================================================
# DOCKER REPOSITORY
# ============================================================

if command -v docker >/dev/null 2>&1; then

    log "Docker already installed."

else

    log "Installing Docker..."

    install -m 0755 -d /etc/apt/keyrings

    curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
        -o /etc/apt/keyrings/docker.asc

    chmod a+r /etc/apt/keyrings/docker.asc

    cat > /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: ${VERSION_CODENAME}
Components: stable
Architectures: amd64
Signed-By: /etc/apt/keyrings/docker.asc
EOF

    apt-get update

    apt-get install -y \
        docker-ce \
        docker-ce-cli \
        containerd.io \
        docker-buildx-plugin \
        docker-compose-plugin

fi


# ============================================================
# DOCKER SERVICE
# ============================================================

systemctl enable docker
systemctl restart docker

usermod -aG docker "${ADMIN_USER}"

log "Docker version:"
docker --version

log "Docker Compose version:"
docker compose version


# ============================================================
# SUPABASE DIRECTORY
# ============================================================

mkdir -p "${SUPABASE_DIR}"

cd "${SUPABASE_DIR}"


# ============================================================
# INSTALL SUPABASE
# ============================================================

if [[ -f "${SUPABASE_DIR}/docker-compose.yml" ]]; then

    log "Supabase already appears to be installed."

else

    log "Installing official Supabase self-hosted Docker stack..."

    curl -fsSL https://supabase.link/setup.sh | sh

fi


# ============================================================
# ENTER SUPABASE DIRECTORY
# ============================================================

cd "${SUPABASE_DIR}"

[[ -f ".env" ]] || die "Supabase .env file was not created."


# ============================================================
# BACKUP ORIGINAL ENV
# ============================================================

cp .env ".env.backup.$(date +%Y%m%d-%H%M%S)"


# ============================================================
# CONFIGURE DOMAIN
# ============================================================

log "Configuring Supabase domain settings..."

set_env() {
    local key="$1"
    local value="$2"

    if grep -qE "^${key}=" .env; then
        sed -i "s|^${key}=.*|${key}=${value}|" .env
    else
        echo "${key}=${value}" >> .env
    fi
}

set_env "SUPABASE_PUBLIC_URL" "https://${API_DOMAIN}"
set_env "API_EXTERNAL_URL" "https://${API_DOMAIN}/auth/v1"
set_env "SITE_URL" "https://${WEB_DOMAIN}"
set_env "PROXY_DOMAIN" "${API_DOMAIN}"


# ============================================================
# GENERATE / CHECK DASHBOARD CREDENTIALS
# ============================================================

if grep -q "^DASHBOARD_USERNAME=" .env; then
    log "Dashboard username already exists."
else
    set_env "DASHBOARD_USERNAME" "admin"
fi

if grep -q "^DASHBOARD_PASSWORD=" .env; then
    log "Dashboard password already exists."
else
    DASHBOARD_PASSWORD=$(openssl rand -base64 32 | tr -dc 'A-Za-z0-9' | head -c 24)
    set_env "DASHBOARD_PASSWORD" "${DASHBOARD_PASSWORD}"

    echo
    echo "============================================================"
    echo " SUPABASE STUDIO CREDENTIALS"
    echo "============================================================"
    echo
    echo "Username: admin"
    echo "Password: ${DASHBOARD_PASSWORD}"
    echo
    echo "SAVE THIS PASSWORD."
    echo
fi


# ============================================================
# ENSURE POSTGRES PORTS ARE NOT PUBLIC
# ============================================================

log "Securing Supabase database ports..."

if [[ -f docker-compose.yml ]]; then

    cp docker-compose.yml \
        "docker-compose.yml.backup.$(date +%Y%m%d-%H%M%S)"

    python3 <<'PY'
from pathlib import Path

p = Path("docker-compose.yml")
text = p.read_text()

old = """    ports:
      - ${POSTGRES_PORT}:5432
      - ${POOLER_PROXY_PORT_TRANSACTION}:6543
"""

if old in text:
    text = text.replace(old, "")
    p.write_text(text)
    print("Removed public PostgreSQL/Supavisor port mappings.")
else:
    print("No matching public PostgreSQL/Supavisor mappings found.")
PY

fi


# ============================================================
# CONFIGURE CADDY
# ============================================================

log "Enabling Supabase Caddy reverse proxy..."

if [[ -f "run.sh" ]]; then

    sh run.sh config add caddy

else

    die "Supabase run.sh not found."

fi


# ============================================================
# CADDYFILE
# ============================================================

CADDYFILE="${SUPABASE_DIR}/volumes/proxy/caddy/Caddyfile"

mkdir -p "$(dirname "${CADDYFILE}")"

if [[ -f "${CADDYFILE}" ]]; then
    cp "${CADDYFILE}" \
        "${CADDYFILE}.backup.$(date +%Y%m%d-%H%M%S)"
fi

cat > "${CADDYFILE}" <<EOF
${API_DOMAIN} {
    @supabase_api path /auth/v1/* /rest/v1/* /graphql/v1 /realtime/v1/* /storage/v1/* /functions/v1/* /mcp /sso/*

    handle @supabase_api {
        reverse_proxy api-gw:8000
    }

    handle {
        respond "Not Found" 404
    }

    header -server
}

${STUDIO_DOMAIN} {
    basic_auth {
        {\$PROXY_AUTH_USERNAME} {\$PROXY_AUTH_PASSWORD}
    }

    reverse_proxy studio:3000

    header -server
}
EOF


# ============================================================
# CHECK DOCKER COMPOSE CONFIG
# ============================================================

log "Validating Docker Compose configuration..."

docker compose config >/dev/null


# ============================================================
# CREATE DATABASE CONTAINER FIRST
# ============================================================

log "Creating Supabase database container..."

docker compose create db


# ============================================================
# START DATABASE
# ============================================================

log "Starting PostgreSQL..."

docker compose up -d db

sleep 10


# ============================================================
# CHECK DATABASE
# ============================================================

log "Checking PostgreSQL health..."

docker compose ps db

if ! docker compose ps db | grep -q "healthy"; then
    warn "PostgreSQL is not healthy yet."
    warn "Waiting another 20 seconds..."
    sleep 20
fi


# ============================================================
# ROTATE DATABASE PASSWORD
# ============================================================

if [[ -f "utils/db-passwd.sh" ]]; then

    log "Rotating Supabase database password..."

    sh utils/db-passwd.sh

else

    warn "db-passwd.sh not found. Skipping DB password rotation."

fi


# ============================================================
# VALIDATE CADDY
# ============================================================

log "Validating Caddy configuration..."

docker compose run --rm --no-deps caddy \
    caddy validate \
    --config /etc/caddy/Caddyfile || true


# ============================================================
# START COMPLETE SUPABASE STACK
# ============================================================

log "Starting complete Supabase stack..."

sh run.sh start


# ============================================================
# RECREATE SUPAVISOR
# ============================================================

log "Recreating Supavisor without public database ports..."

docker compose up -d --force-recreate supavisor


# ============================================================
# RESTART CADDY
# ============================================================

log "Restarting Caddy..."

docker compose restart caddy


# ============================================================
# WAIT FOR SERVICES
# ============================================================

log "Waiting for Supabase services..."

sleep 30


# ============================================================
# SHOW SERVICES
# ============================================================

echo
echo "============================================================"
echo " SUPABASE SERVICES"
echo "============================================================"

docker compose ps


# ============================================================
# CHECK PUBLISHED PORTS
# ============================================================

echo
echo "============================================================"
echo " PUBLIC DOCKER PORTS"
echo "============================================================"

docker compose config --no-interpolate |
    grep -nE 'published:|target:' || true


# ============================================================
# CHECK FIREWALL
# ============================================================

echo
echo "============================================================"
echo " FIREWALL"
echo "============================================================"

ufw status verbose


# ============================================================
# LOCAL CURL TESTS
# ============================================================

echo
echo "============================================================"
echo " HTTPS TESTS"
echo "============================================================"

log "Testing API..."

curl -k -I \
    --max-time 20 \
    "https://${API_DOMAIN}/auth/v1/settings" || true

echo

log "Testing Studio..."

curl -k -I \
    --max-time 20 \
    "https://${STUDIO_DOMAIN}" || true


# ============================================================
# SECURITY CHECK
# ============================================================

echo
echo "============================================================"
echo " DATABASE PORT SECURITY CHECK"
echo "============================================================"

if ss -lntup | grep -E ':5432|:6543' >/dev/null 2>&1; then

    warn "5432 or 6543 appears to be listening on the host."

    ss -lntup | grep -E ':5432|:6543' || true

else

    log "Good: PostgreSQL ports are not publicly listening."

fi


# ============================================================
# FINAL INFORMATION
# ============================================================

echo
echo
echo "============================================================"
echo " SUPABASE INSTALLATION COMPLETE"
echo "============================================================"
echo

echo "Supabase API:"
echo "  https://${API_DOMAIN}"

echo
echo "Supabase Studio:"
echo "  https://${STUDIO_DOMAIN}"

echo
echo "Frontend:"
echo "  https://${WEB_DOMAIN}"

echo
echo "Supabase directory:"
echo "  ${SUPABASE_DIR}"

echo
echo "Docker:"
docker --version

echo
echo "Services:"
docker compose ps

echo
echo "============================================================"
echo " IMPORTANT"
echo "============================================================"
echo
echo "1. Do NOT expose PostgreSQL 5432 publicly."
echo
echo "2. Do NOT expose Supavisor 6543 publicly."
echo
echo "3. Keep service_role / secret keys OFF the frontend."
echo
echo "4. Lovable/frontend should use the Supabase public URL:"
echo
echo "   https://${API_DOMAIN}"
echo
echo "5. Studio:"
echo
echo "   https://${STUDIO_DOMAIN}"
echo
echo "6. Cloudflare SSL should remain:"
echo"
echo "   Full (strict)"
echo
echo "7. Save your Supabase .env securely:"
echo
echo "   ${SUPABASE_DIR}/.env"
echo
echo "============================================================"
echo

if [[ "${AUTO_REBOOT}" == "yes" ]]; then

    warn "Rebooting server in 10 seconds..."
    sleep 10
    reboot

else

    log "Installation finished. No reboot requested."

fi
