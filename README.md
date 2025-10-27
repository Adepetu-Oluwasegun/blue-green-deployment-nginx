
# Blue-Green Deployment with Nginx and Docker Compose on AWS EC2
A production-ready Blue/Green deployment setup with automatic failover using Nginx upstreams. This implementation ensures zero downtime and zero failed client requests during service failures.
---

Live Deployment
This Blue/Green deployment is currently running on AWS EC2:

Public Endpoints:

Main Load Balancer: http://18.212.76.50:8080/version
Blue Service (Direct): http://18.212.76.50:8081/version
Green Service (Direct): http://18.212.76.50:8082/version

## Environment Variables

Create a `.env` file at the root of the project:

```env
# Docker image URLs for blue and green apps
APP_BLUE_IMAGE=<dockerhub_username>/node-app-blue:latest
APP_GREEN_IMAGE=<dockerhub_username>/node-app-green:latest

# Container ports
PORT=8080
---

## Setup Instructions

### Docker Compose Setup

1. Make sure `.env` contains the correct image URLs and ports.

2. Example `docker-compose.yml`:

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:1.24-alpine
    ports:
      - "${PORT}:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app_blue
      - app_green
    restart: unless-stopped

  app_blue:
    image: ${BLUE_IMAGE}
    environment:
      - APP_POOL=blue
      - RELEASE_ID=${RELEASE_ID_BLUE}
    ports:
      - "8081:3000"

  app_green:
    image: ${GREEN_IMAGE}
    environment:
      - APP_POOL=green
      - RELEASE_ID=${RELEASE_ID_GREEN}
    ports:
      - "8082:3000"```

3. Start the deployment:

```bash
docker-compose up -d
```

> Docker Compose will pull the images automatically and start the containers.

---

### Nginx Blue/Green Configuration

1. Nginx configuration (`nginx/nginx.conf.TEMPLATE`):

```nginx
version: '3.8'

services:
  nginx:
    image: nginx:1.24-alpine
    ports:
      - "${PORT}:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app_blue
      - app_green
    restart: unless-stopped

  app_blue:
    image: ${BLUE_IMAGE}
    environment:
      - APP_POOL=blue
      - RELEASE_ID=${RELEASE_ID_BLUE}
    ports:
      - "8081:3000"

  app_green:
    image: ${GREEN_IMAGE}
    environment:
      - APP_POOL=green
      - RELEASE_ID=${RELEASE_ID_GREEN}
    ports:
      - "8082:3000"```

2. Reload Nginx if you update the configuration:

```bash
docker-compose exec nginx nginx -s reload
```

---

### EC2 Deployment

1. SSH into your EC2 instance:

```bash
ssh -i your-key.pem ec2-user@<EC2_PUBLIC_IP>
```

2. Install Docker and Docker Compose:

```bash
sudo apt update
sudo apt install docker.io docker-compose -y
sudo systemctl enable --now docker
```

3. Clone the repository and start the deployment:

```bash
git clone <repo-url>
cd <repo-name>
docker-compose up -d
```

4. Ensure EC2 security groups allow inbound traffic PORT 80

---

## Testing the Deployment

* Access the app: `http://<EC2_PUBLIC_IP>:80`
* Stop one app container and confirm traffic switches to the other app.
* Update the `.env` with a new image tag, then run:

```bash
docker-compose pull
docker-compose up -d
docker-compose exec nginx nginx -s reload
```

This ensures **zero downtime updates**.

---
Chaos Testing:

# Induce errors on Blue
curl -X POST "http://18.212.76.50:8081/chaos/start?mode=error"

# Verify automatic failover
curl "http://18.212.76.50:8080/version"

# Stop chaos
curl -X POST "http://18.212.76.50:8081/chaos/stop"

## Notes

* Using Docker Compose simplifies managing multiple containers.
* Blue/Green deployment allows easy rollback by switching traffic to the previous version.
* Environment variables allow easy updates to image tags without modifying `docker-compose.yml`.

---
