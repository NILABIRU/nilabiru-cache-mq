# Nilabiru Cache & MQ

A self-hosted cache and message broker stack for the Nilabiru ecosystem, bundling Redis and RabbitMQ into a single Docker Compose setup with a simple deploy script.

---

## Overview

**Nilabiru Cache & MQ** provisions and manages an in-memory cache and a message broker — Redis and RabbitMQ — running in isolated Docker containers. Every published port is bound to the server's Tailscale IP, so both services are reachable only over your private Tailscale network. Deployment is handled by a single `deploy.sh` script.

---

## Services

| Service               | Image                            | Port(s)         | Description                                                           |
| --------------------- | -------------------------------- | --------------- | --------------------------------------------------------------------- |
| **nilabiru-redis**    | `redis:8.8-alpine`               | `6379`          | In-memory cache and key-value store with password protection          |
| **nilabiru-rabbitmq** | `rabbitmq:4.1-management-alpine` | `5672`, `15672` | Message broker with management UI available at port `15672`           |

All services run on the default Docker Compose network and use `restart: unless-stopped`. Every published port is bound to the Tailscale IP (`TAILSCALE_IP`) — none are exposed on public network interfaces.

---

## Requirements

- Docker Engine `20.10+`
- Docker Compose `v2+`
- Tailscale installed and connected on the server and all client machines

---

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/NILABIRU/nilabiru-cache-mq.git
cd nilabiru-cache-mq
```

### 2. Configure environment variables

Copy the provided `env` file and fill in all values:

```bash
cp env .env
```

Then edit `.env`:

```env
# Tailscale
TAILSCALE_IP=your_tailscale_ip

# Redis
REDIS_PASSWORD=your_redis_password

# RabbitMQ
RABBITMQ_USER=your_rabbitmq_user
RABBITMQ_PASSWORD=your_rabbitmq_password
RABBITMQ_VHOST=your_rabbitmq_vhost
```

> **Note:** Never commit `.env` to version control. It is already listed in `.gitignore`.

### 3. Start the stack

The recommended way is the provided deploy script:

```bash
chmod +x deploy.sh
./deploy.sh
```

`deploy.sh` stops on the first error (`set -e`) and does the following:

1. Validates the Compose configuration with `docker compose config --quiet`.
2. Deploys/redeploys all services with `docker compose up -d --remove-orphans --build`.
3. Always runs a cleanup on exit (even if a step fails) that removes dangling images with `docker image prune -f`.

Alternatively, you can start the stack directly:

```bash
docker compose up -d
```

To verify all services are running:

```bash
docker compose ps
```

---

## Service Access

All services are accessible only via the Tailscale IP of the server.

| Service             | URL / Address                 |
| ------------------- | ----------------------------- |
| Redis               | `<TAILSCALE_IP>:6379`         |
| RabbitMQ AMQP       | `<TAILSCALE_IP>:5672`         |
| RabbitMQ Management | `http://<TAILSCALE_IP>:15672` |

Example connections:

```bash
# Redis
redis-cli -h <TAILSCALE_IP> -p 6379 -a <REDIS_PASSWORD>

# RabbitMQ (AMQP URL; the default vhost "/" is encoded as %2F)
amqp://<RABBITMQ_USER>:<RABBITMQ_PASSWORD>@<TAILSCALE_IP>:5672/%2F
```

Log in to the RabbitMQ management UI with `RABBITMQ_USER` and `RABBITMQ_PASSWORD`.

---

## Data Persistence

All stateful services use Docker named volumes for reliable persistence across restarts and redeployments.

| Volume          | Type         | Service  | Container path        |
| --------------- | ------------ | -------- | --------------------- |
| `redis-data`    | Named volume | Redis    | `/data`               |
| `rabbitmq-data` | Named volume | RabbitMQ | `/var/lib/rabbitmq`   |

> **Note:** The Redis password is passed on the command line (`--requirepass`), so changing `REDIS_PASSWORD` and redeploying takes effect immediately. RabbitMQ's default user, password, and vhost, however, are only applied on the **first** startup, when its volume is still empty — to change them later, use the management UI or `rabbitmqctl`, or remove the volume (this deletes all queues and messages).

---

## License

This project is licensed under the [MIT License](LICENSE).
Copyright © 2026 Andry Pebrianto