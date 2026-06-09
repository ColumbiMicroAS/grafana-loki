# Loki + Grafana Logging Stack

A Docker-based logging stack with Loki, Grafana, and Promtail. Designed to run alongside Laravel Sail projects and other Docker services. Loki API is protected with basic auth via an nginx reverse proxy.

## Quick Start

### 1. Create the shared network (one-time setup)

```bash
docker network create loki-network
```

### 2. Start the stack

```bash
cd ~/code/grafana-loki
docker compose up -d
```

### 3. Access Grafana

Open [http://localhost:3000](http://localhost:3000) in your browser.

- **Username:** `admin`
- **Password:** `admin`
- Loki is pre-configured as a datasource

## Authentication

The Loki API (port 3100) is protected with **Basic Auth** via an nginx reverse proxy.

| | |
|---|---|
| **Username** | `loki` |
| **Password** | `lokisecret` |

Internal services (Promtail, Grafana) connect to Loki directly on the Docker network and do **not** need credentials.

To change credentials, regenerate `.htpasswd`:
```bash
htpasswd -nbB loki your-new-password > .htpasswd
docker compose restart nginx
```

## Connecting Laravel Sail Projects

### 1. Add the network to your Sail project's `docker-compose.yml`:

```yaml
networks:
  sail:
    driver: bridge
  loki-network:
    external: true

services:
  laravel.test:
    # ... existing config ...
    networks:
      - sail
      - loki-network
```

### 2. Install the Loki logging package:

```bash
composer require itspire/monolog-loki
```

### 3. Add a Loki channel in `config/logging.php`:
```php
   use Itspire\MonologLoki\Handler\LokiHandler;
```
```php
'loki' => [
    'driver' => 'monolog',
    'level' => env('LOG_LEVEL', 'info'),
    'handler' => LokiHandler::class,
    'handler_with' => [
        'apiConfig' => [
            'entrypoint' => 'http://loki-gateway:3100',
            'auth' => [
                'basic' => [
                    env('LOKI_USERNAME', 'loki'),
                    env('LOKI_PASSWORD', 'lokisecret'),
                ],
            ],
        ],
        'labels' => [
            'app' => env('APP_NAME', 'laravel'),
            'env' => env('APP_ENV', 'local'),
        ],
    ],
],
```

### 4. Use it in your code:

In handle() on a job you can use shareContext() to pass information down stream for tagging log lines
```php
Log::shareContext([
  'job' => class_basename($this),
  'client_id' => $this->client->id, // or uid
  'run' => $this->run, // or source run
  'job' =>  class_basename($this),
  // 'task' => '',
  // etc...
]);
```



```php
Log::channel('loki')->info('Order placed', ['order_id' => 123]);
```

Or set as default in `.env`:
```
LOG_CHANNEL=loki
```

## Pushing Logs via API (curl)

From the host machine (requires auth):

```bash
curl -u loki:lokisecret -X POST "http://localhost:3100/loki/api/v1/push" \
  -H "Content-Type: application/json" \
  -d '{
    "streams": [{
      "stream": { "app": "my-app", "env": "local" },
      "values": [
        ["'"$(date +%s)000000000"'", "Hello from my app!"]
      ]
    }]
  }'
```

From inside a Docker container on the `loki-network` (via gateway, requires auth):

```bash
curl -u loki:lokisecret -X POST "http://loki-gateway:3100/loki/api/v1/push" \
  -H "Content-Type: application/json" \
  -d '{
    "streams": [{
      "stream": { "app": "my-app", "env": "local" },
      "values": [
        ["'"$(date +%s)000000000"'", "Hello from container!"]
      ]
    }]
  }'
```

## Querying Logs via API

```bash
# Query all logs from the last hour
curl -u loki:lokisecret -G "http://localhost:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={app="my-app"}' \
  --data-urlencode "start=$(date -d '1 hour ago' +%s)000000000" \
  --data-urlencode "end=$(date +%s)000000000"

# Query logs by compose project (auto-collected by Promtail)
curl -u loki:lokisecret -G "http://localhost:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={compose_project="netvision-payroll-plus"}'
```

## Auto-Collected Logs (Promtail)

Promtail automatically discovers all Docker containers and collects their logs. Labels available in Grafana:

| Label | Description |
|-------|-------------|
| `container` | Container name |
| `compose_project` | Docker Compose project name |
| `compose_service` | Docker Compose service name |
| `container_id` | Container ID |

## Services & Ports

| Service | Port | Purpose |
|---------|------|---------|
| Grafana | 3000 | Web UI for log exploration |
| Nginx (loki-gateway) | 3100 | Authenticated reverse proxy to Loki |
| Loki | internal | Log storage (not exposed externally) |
| Promtail | internal | Agent collecting Docker logs |

## Managing the Stack

```bash
# Start
docker compose up -d

# Stop
docker compose down

# View logs
docker compose logs -f

# Reset all data
docker compose down -v
```

## Troubleshooting

### 401 Unauthorized
Make sure you're passing credentials: `-u loki:lokisecret` or the `basicAuth` config in Laravel.

### Port conflicts
If port 3000 or 3100 is in use, edit `docker-compose.yml` and change the host port (left side of `:`).

### Network not found
If you see `network loki-network declared as external, but could not be found`:
```bash
docker network create loki-network
```

### Sail containers can't reach Loki
Make sure both the Sail project and this stack are using the `loki-network`. Restart both after changes:
```bash
# In this directory
docker compose restart

# In your Sail project
./vendor/bin/sail down && ./vendor/bin/sail up -d
```

## Displaying logs

### Laravel

Check out the branch [loki-logging](https://github.com/ColumbiMicroAS/netvision-payroll-plus/tree/feature/loki-logging) on netvision-payroll-plus repo. 

Frontend: [LogsIndex.vue](https://github.com/ColumbiMicroAS/netvision-payroll-plus/blob/feature/loki-logging/resources/js/Pages/Admin/Logs/LogsIndex.vue)
Backend: [AdminLogsController.php](https://github.com/ColumbiMicroAS/netvision-payroll-plus/blob/feature/loki-logging/app/Http/Controllers/Admin/AdminLogsController.php)



