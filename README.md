
# Blue-Green Deployment with Nginx and Docker Compose on AWS EC2

## Project Structure

```
project-root/
├─ dockerfile.nginx
├─ entrypoint.sh           # entrypoint script 
├─ .env.example            # Environment variables for Docker images, versions and port
├─ nginx.conf.template        # config for blue/green deployment
├─ docker-compose.yml         # Compose file for blue/green deployment
└─ README.md
```

---

## Environment Variables

Create a `.env` file at the root of the project:

```env
# Docker image URLs for blue and green apps
APP_BLUE_IMAGE=<dockerhub_username>/node-app-blue:latest
APP_GREEN_IMAGE=<dockerhub_username>/node-app-green:latest

# Container ports
BLUE_PORT=8001
GREEN_PORT=8002
NGINX_PORT=8080
```

---

## Setup Instructions

### Docker Compose Setup

1. Make sure `.env` contains the correct image URLs and ports.

2. Example `docker-compose.yml`:

```yaml
version: "3.8"

services:
  app_blue:
    image: ${APP_BLUE_IMAGE}
    container_name: app_blue
    ports:
      - "${BLUE_PORT}: ${PORT}"

  app_green:
    image: ${APP_GREEN_IMAGE}
    container_name: app_green
    ports:
      - "${GREEN_PORT}:${PORT}"

  nginx:
    image: nginx
    container_name: nginx
    ports:
      - "${NGINX_PORT}:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app_blue
      - app_green
```

3. Start the deployment:

```bash
docker-compose up -d
```

> Docker Compose will pull the images automatically and start the containers.

---

### Nginx Blue/Green Configuration

1. Nginx configuration (`nginx/nginx.conf.TEMPLATE`):

```nginx
http {
    upstream backend {
        server app_blue:${PORT} max_fails=1 fail_timeout=30s;
        server app_green:${PORT} max_fails=1 fail_timeout=30s;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

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

4. Ensure EC2 security groups allow inbound traffic on `$NGINX_PORT` (e.g., 8080).

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

## Notes

* Using Docker Compose simplifies managing multiple containers.
* Blue/Green deployment allows easy rollback by switching traffic to the previous version.
* Environment variables allow easy updates to image tags without modifying `docker-compose.yml`.

---
