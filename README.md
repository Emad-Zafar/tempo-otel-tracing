# Distributed tracing with Tempo, OpenTelemetry and Grafana

Follow a single request as it moves across your services. Your apps send traces
to an OpenTelemetry Collector, the collector forwards them to Tempo, and Grafana
shows them. Local disk out of the box, S3 when you need it.

Tracing needs a little work in your app: you attach the OpenTelemetry agent or
SDK so it emits spans. The full walkthrough for Spring Boot and Node.js is in
[docs/INSTRUMENTATION.md](docs/INSTRUMENTATION.md).

## What you get

- One OTLP endpoint your services send traces to
- Tempo storing traces, searchable with TraceQL
- Grafana with Tempo connected and per trace node graphs on
- A clear instrumentation guide for Java and Node.js

## Good for

More than one service, where you want to see a request cross service boundaries
and find where time goes. Works on Compose or plain containers.

## Run it

This stack shares a Docker network with your app containers so they can reach
the collector. Create it once, then start up:

```bash
docker network create observability-net
cp .env.example .env       # set a Grafana password
docker compose up -d
```

Then instrument at least one service and point it at the collector (see the
guide). Send some traffic and open Grafana at `http://localhost:3000`, go to
Explore, pick Tempo, and run:

```
{ resource.service.name = "my-service" }
```

## Send traces from your app

Put your service on `observability-net` and export OTLP to the collector:

```yaml
services:
  my-service:
    environment:
      - OTEL_SERVICE_NAME=my-service
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
    networks:
      - observability-net

networks:
  observability-net:
    external: true
```

How to attach the Java agent, load the Node.js tracing file, and get trace ids
into your logs is all in [docs/INSTRUMENTATION.md](docs/INSTRUMENTATION.md).

## Store traces in S3

Local disk is fine for dev. For durable storage:

1. Create a bucket and let the host write to it.
2. Set `TEMPO_S3_BUCKET_NAME` and `TEMPO_S3_REGION` in `.env`.
3. Point Tempo at the S3 config in `docker-compose.yml`:

   ```yaml
   - ./tempo/tempo.s3.yaml:/etc/tempo/tempo.yaml:ro
   ```

4. Recreate it: `docker compose up -d --force-recreate tempo`

## Service graphs

Grafana draws a node graph for a single trace out of the box. The aggregated
service graph (which service calls which, with rates and errors) needs a
Prometheus to hold the generated metrics. That is wired in the full
observability stack. The Tempo config here leaves it ready with steps commented.

## Ports

Grafana 3000, Tempo 3200, collector OTLP 4317 and 4318.

## Troubleshooting

- No traces? Watch `docker compose logs otel-collector`. Received spans with an
  export error means Tempo is unreachable. Nothing received means the app is not
  reaching the collector, usually a port, protocol or network problem.
- gRPC is 4317, HTTP is 4318. The protocol setting has to match the port.
- Trace cut off mid request? A proxy is stripping the `traceparent` header.

## License

MIT, see [LICENSE](LICENSE).
