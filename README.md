# Distributed tracing with Tempo, OpenTelemetry and Grafana

A self-contained tracing stack. Your services send traces to an OpenTelemetry
Collector, the collector forwards them to Tempo, and Grafana shows them. Storage
is local disk by default with a documented switch to Amazon S3.

Unlike logs and container metrics, tracing needs a little work inside your app:
you attach the OpenTelemetry agent or SDK so it emits spans. The full walk
through for Spring Boot and Node.js is in [docs/INSTRUMENTATION.md](docs/INSTRUMENTATION.md).

## What you get

- One OTLP endpoint your services send traces to
- Tempo storing traces, queryable with TraceQL
- Grafana with the Tempo data source connected and per trace node graphs on
- A clear instrumentation guide for Java and Node.js, including trace ids in logs

## Which setups this fits

Use this when:

- You have more than one service and want to follow a single request as it moves
  between them.
- You want to see where time goes inside a request, across HTTP calls, database
  queries and outgoing calls.
- You are on Docker Compose or plain containers on a host.

Tracing earns its keep once requests cross service boundaries. For a single
small service it is still useful for spotting slow database calls, but the
payoff grows with the number of services.

## What's inside

```
docker-compose.yml               three services
tempo/tempo.yaml                 Tempo, local disk storage (default)
tempo/tempo.s3.yaml              Tempo, S3 storage (swap in when needed)
otel-collector/config.yaml       receives OTLP, forwards to Tempo
grafana/provisioning/...         Tempo wired in as a data source
docs/INSTRUMENTATION.md          how to instrument your services
.env.example                     copy to .env and edit
```

## Requirements

- Docker and Docker Compose
- Ports 3000 (Grafana), 3200 (Tempo), 4317 and 4318 (collector OTLP) free

## Quick start

This stack joins a shared Docker network called `observability-net` so your app
containers can reach the collector by name. Create it once:

```bash
docker network create observability-net
```

Then:

1. Copy the environment file and set a Grafana password.

   ```bash
   cp .env.example .env
   ```

2. Start the stack.

   ```bash
   docker compose up -d
   ```

3. Instrument at least one service and point it at the collector. Follow
   [docs/INSTRUMENTATION.md](docs/INSTRUMENTATION.md). In short, your app sends
   OTLP to `otel-collector:4317` over the `observability-net` network.

4. Send some traffic, then open Grafana at `http://localhost:3000`, go to
   Explore, pick Tempo, and run:

   ```
   { resource.service.name = "my-service" }
   ```

## Send traces from your app

Your service needs to be on the `observability-net` network and configured to
export OTLP to the collector. Attach it in its own compose file:

```yaml
services:
  my-service:
    # ... your image and config ...
    environment:
      - OTEL_SERVICE_NAME=my-service
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
    networks:
      - observability-net

networks:
  observability-net:
    external: true
```

The language specific details, including how to attach the Java agent, how to
load the Node.js tracing file, and how to get trace ids into your logs, are in
[docs/INSTRUMENTATION.md](docs/INSTRUMENTATION.md).

## How the pieces connect

```
your services  ->  OpenTelemetry Collector  ->  Tempo  ->  Grafana
  (send OTLP)       (receive, batch,             (store    (search and
                     forward)                     traces)   view traces)
```

The collector sits in the middle on purpose. Your apps only ever know one
endpoint, and you can change batching, sampling or where traces are sent by
editing the collector, not every service.

## Store traces in S3

Local disk is fine for development. For durable storage, point Tempo at S3.

1. Create a bucket, for example `my-project-tempo-traces`.

2. Give the host or container permission to read and write it. On EC2 use an
   instance role scoped to the bucket. Off EC2, mount credentials into the Tempo
   container:

   ```yaml
   # under the tempo service in docker-compose.yml
   volumes:
     - ~/.aws:/root/.aws:ro
   environment:
     - AWS_SDK_LOAD_CONFIG=1
     - AWS_PROFILE=default
   ```

3. Set the bucket and region in `.env`:

   ```
   TEMPO_S3_BUCKET_NAME=my-project-tempo-traces
   TEMPO_S3_REGION=us-east-1
   ```

4. Point the Tempo service at the S3 config in `docker-compose.yml`:

   ```yaml
   - ./tempo/tempo.s3.yaml:/etc/tempo/tempo.yaml:ro
   ```

5. Recreate Tempo and watch for storage errors:

   ```bash
   docker compose up -d --force-recreate tempo
   docker compose logs -f tempo
   ```

## Service graphs

Grafana draws a node graph for a single trace out of the box. The aggregated
service graph, the map of which service calls which with request and error
rates, is built by Tempo's metrics generator and needs a Prometheus to store the
generated metrics. That is wired in the full observability stack. The Tempo
config here leaves the generator ready with the steps commented, so you can turn
it on if you add a Prometheus.

## Common settings

- Retention: `block_retention` under `compactor` in the Tempo config, default
  168h (7 days).
- Sampling: set on the app side. See the do's and don'ts in the instrumentation
  guide.
- Grafana login: `GRAFANA_ADMIN_USER` and `GRAFANA_ADMIN_PASSWORD` in `.env`.

## Troubleshooting

- No traces in Tempo. Watch `docker compose logs otel-collector`. If it shows
  received spans but an export error, Tempo is unreachable or misconfigured. If
  it shows nothing, the app is not reaching the collector, usually a port,
  protocol or network problem.
- Traces cut off between services. A proxy is stripping the `traceparent` header.
- Grafana shows the data source but no traces. Confirm the service name in your
  TraceQL query matches `OTEL_SERVICE_NAME`.

## License

MIT, see [LICENSE](LICENSE).
