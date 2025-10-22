#!/bin/sh

# Automated Deployment Script
# POSIX-compliant batch script for git deployment via SSH

set -e

# Color codes for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Logging functions
log_info() { printf "${BLUE}[INFO]${NC} %s\n" "$1"; }
log_success() { printf "${GREEN}[SUCCESS]${NC} %s\n" "$1"; }
log_warning() { printf "${YELLOW}[WARNING]${NC} %s\n" "$1"; }
log_error() { printf "${RED}[ERROR]${NC} %s\n" "$1" >&2; }

# Input collection with validation
get_user_input() {
    # Git Repository URL
    while true; do
        printf "Enter Git repository URL: "
        read -r GIT_REPO_URL
        if echo "$GIT_REPO_URL" | grep -Eq '^(https?|git|ssh)://|^git@[^:]+:[^/]+/'; then
            break
        else
            log_error "Invalid git repository URL format"
        fi
    done

    # Personal Access Token
    while true; do
        printf "Enter Personal Access Token: "
        stty -echo
        read -r GIT_TOKEN
        stty echo
        printf "\n"
        [ -n "$GIT_TOKEN" ] && break
        log_error "Token cannot be empty"
    done

    # Server details
    printf "Enter SSH username: "
    read -r SSH_USERNAME

    while true; do
        printf "Enter Server IP address: "
        read -r SERVER_IP
        echo "$SERVER_IP" | grep -Eq '^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$' && break
        log_error "Invalid IP address format"
    done

    while true; do
        printf "Enter SSH key path: "
        read -r SSH_KEY_PATH
        [ -f "$SSH_KEY_PATH" ] && [ -r "$SSH_KEY_PATH" ] && break
        log_error "SSH key file not found or not readable: $SSH_KEY_PATH"
    done

    # Application port
    while true; do
        printf "Enter application port: "
        read -r APP_PORT
        echo "$APP_PORT" | grep -Eq '^[0-9]+$' && [ "$APP_PORT" -ge 1 ] && [ "$APP_PORT" -le 65535 ] && break
        log_error "Invalid port number. Must be between 1 and 65535"
    done

    # Domain name
    while true; do
        printf "Enter domain name: "
        read -r DOMAIN_NAME
        echo "$DOMAIN_NAME" | grep -Eq '^[a-zA-Z0-9]([a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?(\.[a-zA-Z0-9]([a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)*$' && break
        log_error "Invalid domain name format"
    done

    printf "Enter Git branch (default: main): "
    read -r GIT_BRANCH
    GIT_BRANCH=${GIT_BRANCH:-main}
}

# Setup repository
setup_repository() {
    local repo_name=$(basename "$GIT_REPO_URL" .git)
    
    log_info "Setting up repository: $repo_name"
    
    if [ -d "$repo_name" ]; then
        log_info "Repository exists, updating..."
        cd "$repo_name"
        git remote set-url origin "https://oauth2:${GIT_TOKEN}@${GIT_REPO_URL#https://}"
        git pull origin "$GIT_BRANCH" || { log_error "Failed to pull changes"; return 1; }
        log_success "Repository updated"
    else
        log_info "Cloning repository..."
        local auth_url="https://oauth2:${GIT_TOKEN}@${GIT_REPO_URL#https://}"
        git clone -b "$GIT_BRANCH" "$auth_url" || { log_error "Failed to clone repository"; return 1; }
        cd "$repo_name"
        log_success "Repository cloned"
    fi

    [ -f "Dockerfile" ] || { log_error "Dockerfile not found"; return 1; }
    log_success "Dockerfile verified"
    return 0
}

# Test SSH connection
test_ssh() {
    log_info "Testing SSH connection..."
    ssh -o ConnectTimeout=10 -o BatchMode=yes -o StrictHostKeyChecking=no \
        -i "$SSH_KEY_PATH" "${SSH_USERNAME}@${SERVER_IP}" "echo 'SSH connection successful'" ||
        { log_error "SSH connection failed"; return 1; }
    return 0
}

# Deploy to remote
deploy_remote() {
    local repo_name=$(basename "$(pwd)")
    
    log_info "Starting remote deployment..."
    
    # Create remote deployment script
    cat > remote_deploy.sh << EOF
#!/bin/sh
set -e

echo "[REMOTE] Starting deployment..."

# Cleanup existing containers
docker stop ${repo_name}-container 2>/dev/null || true
docker rm ${repo_name}-container 2>/dev/null || true

# Build and run new container
docker build -t ${repo_name}:latest .
docker run -d --name ${repo_name}-container -p 127.0.0.1:${APP_PORT}:${APP_PORT} ${repo_name}:latest

echo "[REMOTE] Container deployed on port ${APP_PORT}"
EOF

    # Create nginx config
    cat > nginx.conf << EOF
server {
    listen 80;
    server_name ${DOMAIN_NAME} www.${DOMAIN_NAME};
    
    location / {
        proxy_pass http://localhost:${APP_PORT};
        proxy_set_header Host \\$host;
        proxy_set_header X-Real-IP \\$remote_addr;
        proxy_set_header X-Forwarded-For \\$proxy_add_x_forwarded_for;
    }
    
    location ^~ /.well-known/acme-challenge/ {
        root /var/www/html;
    }
}
EOF

    # Copy and execute remotely
    scp -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no \
        remote_deploy.sh nginx.conf \
        "${SSH_USERNAME}@${SERVER_IP}:/tmp/" || { log_error "Failed to copy files"; return 1; }

    # Execute deployment
    ssh -o StrictHostKeyChecking=no -i "$SSH_KEY_PATH" "${SSH_USERNAME}@${SERVER_IP}" "
        cd /tmp && sh remote_deploy.sh &&
        sudo cp nginx.conf /etc/nginx/sites-available/${repo_name} &&
        sudo ln -sf /etc/nginx/sites-available/${repo_name} /etc/nginx/sites-enabled/ &&
        sudo nginx -t && sudo systemctl reload nginx &&
        echo '[REMOTE] Nginx configured successfully'
    " || { log_error "Remote deployment failed"; return 1; }

    # Cleanup local temp files
    rm -f remote_deploy.sh nginx.conf
    
    return 0
}

# Main execution
main() {
    log_info "Starting automated deployment..."
    
    get_user_input
    setup_repository
    test_ssh
    deploy_remote
    
    log_success "Deployment completed successfully!"
    log_info "Your application is now available at: http://${DOMAIN_NAME}"
}

main "$@"
