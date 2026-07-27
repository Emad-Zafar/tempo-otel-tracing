# Distributed tracing with Tempo, Alloy and Grafana

Follow a single request as it moves across your services. Your apps send traces
over OTLP to Grafana Alloy, Alloy forwards them to Tempo, and Grafana shows
them. Local disk out of the box, S3 when you need it.

Alloy is the collector here, the same agent the full observability stack uses,
so this repo is the tracing slice of that stack. Tracing needs a little work in
your app: you attach the OpenTelemetry agent or SDK so it emits spans. The full
walkthrough for Spring Boot and Node.js is in
[docs/INSTRUMENTATION.md](docs/INSTRUMENTATION.md).

## What you get

- One OTLP endpoint (Alloy) your services send traces to
- Tempo storing traces, searchable with TraceQL
- Grafana with Tempo connected and per trace node graphs on
- An instrumentation guide for Java and Node.js

## Good for

More than one service, where you want to see a request cross service boundaries
and find where time goes. Works on Compose or plain containers.

## The stack

| Service | Image | Port |
| --- | --- | --- |
| Grafana | grafana/grafana:12.4.2 | 3030 |
| Tempo | grafana/tempo:2.10.3 | 3200 |
| Alloy | grafana/alloy:latest | 12345, 4317, 4318 |

```
your services  ->  Alloy (OTLP receiver)  ->  Tempo  ->  Grafana
   send OTLP        batch and forward          store      search traces
```

## Run it

This stack shares a Docker network with your app containers so they can reach
Alloy. Create it once, then start up:

```bash
docker network create observability-net
cp .env.example .env
docker compose up -d
```

Instrument at least one service and point it at Alloy (see the guide). Send some
traffic, then open Grafana at `http://localhost:3030` (no login), go to Explore,
pick Tempo, and search by service name.

## Send traces from your app

Put your service on `observability-net` and export OTLP to Alloy:

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

How to attach the Java agent, load the Node.js tracing file, and get trace ids
into your logs is all in [docs/INSTRUMENTATION.md](docs/INSTRUMENTATION.md).

## Store traces in S3

Local disk is fine for dev. For durable storage:

1. Create a bucket and let the host write to it (an EC2 instance role, or mount
   credentials into the Tempo container).
2. Set `TEMPO_S3_BUCKET_NAME` and `TEMPO_S3_REGION` in `.env`.
3. Point Tempo at the S3 config in `docker-compose.yml`:

   ```yaml
   - ./tempo/tempo-config.s3.yml:/etc/tempo.yaml
   ```

4. Recreate it: `docker compose up -d --force-recreate tempo`

## Service graphs

Grafana draws a node graph for a single trace out of the box. The aggregated
service graph (which service calls which, with rates and errors) needs a
Prometheus to hold the metrics Tempo generates from spans. That is wired in the
full observability stack.

## Ports

Grafana 3030, Tempo 3200, Alloy UI 12345, Alloy OTLP 4317 and 4318.

## Troubleshooting

- No traces? Watch `docker compose logs alloy` and check the
  `otelcol.receiver.otlp` component at `http://localhost:12345`. Nothing received
  means the app is not reaching Alloy, usually a port, protocol or network
  problem.
- gRPC is 4317, HTTP is 4318. The protocol setting has to match the port.
- Trace cut off mid request? A proxy is stripping the `traceparent` header.

## License

MIT, see [LICENSE](LICENSE).
