# Distributed tracing with Tempo, Alloy and Grafana

Follow a single request as it moves across your services. Your apps send traces
over OTLP to Grafana Alloy, Alloy forwards them to Tempo, and Grafana shows them.
Local disk out of the box, S3 when you need it.

Alloy is the collector here, the same agent the full observability stack uses,
so this repo is the tracing slice of that stack.

This README is a build-it-yourself guide. Follow the steps to add tracing to
your own project by copying each file in order. If you just want it running,
clone this repo and skip to [Run it](#run-it). Instrumenting your services is a
separate step covered in [docs/INSTRUMENTATION.md](docs/INSTRUMENTATION.md).

## Good for

More than one service, where you want to see a request cross service boundaries
and find where time goes. Works on Compose or plain containers.

## Prerequisites

- Docker and Docker Compose on the host
- Ports 3030 (Grafana), 3200 (Tempo), 4317 and 4318 (Alloy OTLP), 12345 (Alloy
  UI) free

## What you will create

```
your-project/
  tempo/
    tempo-config.yml
  alloy/
    config.alloy
  grafana/provisioning/datasources/
    tempo.yml
  .env
  docker-compose.yml
```

Build the files in the order below. Each step says what the file does and which
file in this repo to copy it from.

## Step 1: Tempo storage config

Create `tempo/tempo-config.yml`. Copy it from
[`tempo/tempo-config.yml`](tempo/tempo-config.yml).

The trace store. It receives OTLP on 4317 and 4318, keeps traces on local disk,
and holds them for 48 hours (`block_retention`). This is where Alloy sends
traces in the next step. Leave it as is.

## Step 2: Alloy collector config

Create `alloy/config.alloy`. Copy it from [`alloy/config.alloy`](alloy/config.alloy).

This receives OTLP traces from your services on `alloy:4317` and `alloy:4318`,
batches them, and forwards to `tempo:4317`. Your apps only ever know the Alloy
address, so you change batching or sampling here, not in every service. Leave it
as is.

## Step 3: Grafana data source

Create `grafana/provisioning/datasources/tempo.yml`. Copy it from
[`grafana/provisioning/datasources/tempo.yml`](grafana/provisioning/datasources/tempo.yml).

Tells Grafana about Tempo on first boot, pointing at `http://tempo:3200`, with
the per trace node graph enabled. Leave it as is.

## Step 4: Environment file

Copy [`.env.example`](.env.example) to `.env`:

```bash
cp .env.example .env
```

Grafana runs with anonymous access here, so there is no password to set. The
only values in the file are for switching Tempo to S3 later.

## Step 5: Compose file

Create `docker-compose.yml`. Copy it from
[`docker-compose.yml`](docker-compose.yml).

Last, because it ties everything together. It runs Grafana, Tempo and Alloy, and
mounts each config you created. It joins an external Docker network called
`observability-net` so your app containers can reach Alloy. You should not need
to edit this.

## Run it

Create the shared network first, then start up:

```bash
docker network create observability-net
docker compose up -d
```

## Step 6: Instrument a service

Traces do not exist until your code emits them. Open
[docs/INSTRUMENTATION.md](docs/INSTRUMENTATION.md) and follow the Spring Boot or
Node.js steps. In short, put your service on `observability-net` and export OTLP
to Alloy:

```yaml
services:
  my-service:
    environment:
      - OTEL_SERVICE_NAME=my-service
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://alloy:4317
    networks:
      - observability-net

networks:
  observability-net:
    external: true
```

Send some traffic, then open Grafana at `http://localhost:3030` (no login), go
to Explore, pick Tempo, and search by service name.

## Store traces in S3

Local disk is fine for dev. For durable storage:

1. Create a bucket and let the host write to it.
2. Set `TEMPO_S3_BUCKET_NAME` and `TEMPO_S3_REGION` in `.env`.
3. In `docker-compose.yml`, change the Tempo config mount to the S3 version:

   ```yaml
   - ./tempo/tempo-config.s3.yml:/etc/tempo.yaml
   ```

   That file is [`tempo/tempo-config.s3.yml`](tempo/tempo-config.s3.yml).
4. Recreate Tempo: `docker compose up -d --force-recreate tempo`

## Service graphs

Grafana draws a node graph for a single trace out of the box. The aggregated
service graph (which service calls which, with rates and errors) needs a
Prometheus to hold the metrics Tempo generates from spans. That is wired in the
full observability stack.

## Troubleshooting

- No traces? Watch `docker compose logs alloy` and check the
  `otelcol.receiver.otlp` component at `http://localhost:12345`. Nothing received
  means the app is not reaching Alloy, usually a port, protocol or network
  problem.
- gRPC is 4317, HTTP is 4318. The protocol setting has to match the port.
- Trace cut off mid request? A proxy is stripping the `traceparent` header.

## License

MIT, see [LICENSE](LICENSE).
