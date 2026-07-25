# Instrumenting your services for tracing

This guide shows how to make a Spring Boot service and a Node.js service emit
traces to this stack, and how to get the trace id into your logs so you can jump
between a log line and the trace it came from. It is written to be copied step
by step on the next project.

The short version:

- Traces go from your app to the OpenTelemetry Collector over OTLP, and the
  collector forwards them to Tempo.
- For Java you attach the OpenTelemetry Java agent. No code changes.
- For Node.js you load a small tracing file before your app starts.
- Both auto-instrument the common libraries (HTTP, database clients, and so on),
  and both can inject the trace id into your logs.

## Where data goes

Your app sends to the collector. In this stack the collector listens on:

- `otel-collector:4317` for OTLP over gRPC
- `otel-collector:4318` for OTLP over HTTP

Use the hostname `otel-collector` when your app runs as a container on the same
Docker network. Use `http://<host-ip>:4317` when your app runs outside Docker on
the same machine. Pick one protocol and keep the port and protocol matched. gRPC
uses 4317, HTTP uses 4318. A mismatched pair is the most common reason nothing
shows up.

## Java and Spring Boot

### 1. Get the agent into the image

Download the agent jar and copy it into your container. In your Dockerfile:

```dockerfile
ADD https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar /opentelemetry-javaagent.jar
```

Pin a specific version in production rather than `latest` so builds are
repeatable. Check the releases page and use a tagged download URL.

### 2. Start the JVM with the agent

```dockerfile
ENTRYPOINT ["java", \
  "-javaagent:/opentelemetry-javaagent.jar", \
  "-jar", "my-service.jar"]
```

Configure the agent with environment variables. These are the ones that matter:

```
OTEL_SERVICE_NAME=my-service
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
OTEL_METRICS_EXPORTER=none
OTEL_LOGS_EXPORTER=none
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production
```

Notes:

- `OTEL_SERVICE_NAME` is the name you will see in Tempo and in the service
  graph. Use the same name for every instance of the service.
- Set `OTEL_METRICS_EXPORTER=none` and `OTEL_LOGS_EXPORTER=none` if you only want
  traces from the agent. Leave metrics to Prometheus and logs to your log
  collector.
- `deployment.environment` lets you tell staging from production in the same
  Tempo. Add anything else useful, comma separated.

If you would rather send over HTTP, set the endpoint to
`http://otel-collector:4318` and `OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf`.

### 3. Put the trace id in your logs

The Java agent writes `trace_id`, `span_id` and `trace_flags` into the logging
context (MDC) for every log line inside a traced request. You just have to print
them.

For Log4j2, add them to your pattern. The key part is printing `trace_id=` so
the log collector can pick it out with a simple pattern:

```xml
<PatternLayout pattern="%d{ISO8601} %-5level trace_id=%X{trace_id} span_id=%X{span_id} %logger{36} - %msg%n"/>
```

For Logback the same idea:

```xml
<pattern>%d{ISO8601} %-5level trace_id=%X{trace_id} span_id=%X{span_id} %logger{36} - %msg%n</pattern>
```

If you log JSON with the Log4j2 JsonTemplateLayout, add the fields there
instead so each record carries a `trace_id` field:

```xml
<JsonTemplateLayout eventTemplateUri="classpath:EcsLayout.json">
  <EventTemplateAdditionalField key="trace_id" format="pattern" value="%X{trace_id}"/>
  <EventTemplateAdditionalField key="span_id" format="pattern" value="%X{span_id}"/>
</JsonTemplateLayout>
```

Lines logged outside a request (startup, background jobs with no parent trace)
will have an empty trace_id. That is expected.

### 4. What you get for free

The agent instruments Spring MVC and WebFlux, RestTemplate and WebClient, JDBC
and MongoDB drivers, Kafka, and common AWS SDK calls, among others. Outgoing
HTTP and messaging calls carry the trace context automatically, so a request
that crosses several services shows up as one trace.

## Node.js

### 1. Install the packages

```bash
npm install @opentelemetry/sdk-node \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-trace-otlp-grpc
```

### 2. Create a tracing file

Create `tracing.js` at the root of your service:

```js
const { NodeSDK } = require("@opentelemetry/sdk-node");
const { getNodeAutoInstrumentations } = require("@opentelemetry/auto-instrumentations-node");
const { OTLPTraceExporter } = require("@opentelemetry/exporter-trace-otlp-grpc");

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter(),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();

process.on("SIGTERM", () => {
  sdk.shutdown().finally(() => process.exit(0));
});
```

The exporter and service name come from environment variables, so you do not
hard code them here:

```
OTEL_SERVICE_NAME=my-node-service
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
```

### 3. Load it before your app

The tracing file has to run before anything else so it can patch the libraries
your app uses. Load it with the require flag:

```json
{
  "scripts": {
    "start": "node -r ./tracing.js index.js"
  }
}
```

Or in the Dockerfile:

```dockerfile
CMD ["node", "-r", "./tracing.js", "index.js"]
```

Loading it inside your app with a normal `require` in the middle of the file is
too late and will miss instrumentation.

### 4. Put the trace id in your logs

`getNodeAutoInstrumentations` includes the Winston and Pino instrumentations.
When a logger runs inside a traced request, they add `trace_id`, `span_id` and
`trace_flags` to the log record on their own. You do not add any code, you only
make sure the fields end up in the output.

With a Winston JSON logger the fields appear as top level keys:

```js
const { createLogger, format, transports } = require("winston");

const logger = createLogger({
  format: format.combine(format.timestamp(), format.json()),
  transports: [new transports.Console()],
});

module.exports = logger;
```

A line then looks like this, with the ids injected for you:

```json
{"level":"info","message":"charge captured","timestamp":"2026-01-01T00:00:00Z","trace_id":"4bf9...","span_id":"00f0..."}
```

Pino works the same way and already outputs JSON, so the ids show up as fields
without extra work.

If you use a plain text log format instead of JSON, print the id yourself so the
collector can find it, for example by including `trace_id=` and the value in the
message pattern. JSON is easier because Loki can parse it directly.

## Confirm it is working

1. Send some traffic through your service.
2. Watch the collector. It logs a line per batch when the debug exporter is on:

   ```bash
   docker compose logs -f otel-collector
   ```

3. In Grafana open Explore, pick Tempo, and run a TraceQL query such as
   `{ resource.service.name = "my-service" }`. You should see traces.

If the collector shows nothing, the problem is between your app and the
collector, usually the endpoint, the protocol, or the network. If the collector
receives spans but Tempo has none, check the collector logs for an export error.

## Correlating logs and traces in Grafana

Once trace ids are in your logs and your traces are in Tempo, Grafana can link
the two in both directions:

- Log to trace. On the Loki data source you define a derived field that reads
  the `trace_id` from a log line and turns it into a link to Tempo. Click the id
  on a log line and you land on the trace.
- Trace to logs. On the Tempo data source you point trace to logs at Loki and
  tell it to match on `trace_id`. Open a span and jump to the logs from that
  moment.

Both directions need a Loki data source present, so they are pre-wired in the
full observability stack rather than in this tracing only stack. This repo gives
you the tracing half. When you are ready for the round trip, use the full stack.

## Service graphs and node graphs

- Node graph. Grafana draws a node graph for a single trace from its spans with
  no extra setup. It is already enabled on the Tempo data source here.
- Service graph. The aggregated map of which service calls which, with request
  rates and error rates, is built by Tempo's metrics generator. It turns spans
  into metrics and writes them to a Prometheus, and Grafana reads that back. That
  needs a Prometheus, so it is wired in the full observability stack. The Tempo
  config here has the generator block ready with the steps commented.

## Do's and don'ts

Do:

- Use one service name per service, the same across every instance.
- Set `deployment.environment` so staging and production do not blur together.
- Sample when volume is high. `OTEL_TRACES_SAMPLER=parentbased_traceidratio` with
  `OTEL_TRACES_SAMPLER_ARG=0.1` keeps one in ten traces and keeps whole traces
  intact.
- Keep the trace context flowing. Do not strip `traceparent` headers at proxies
  or gateways, or traces break at that hop.
- Print the trace id in logs from day one. It costs nothing and saves you later.

Don't:

- Do not put high cardinality values such as user ids or emails into span
  attributes or into log labels. It blows up storage and index size.
- Do not use a different service name per pod or per instance. You lose the
  aggregate view.
- Do not block startup waiting for the collector. The SDK buffers and retries, so
  a collector that is briefly down does not take your app down.
- Do not log secrets. Auto-instrumentation can capture headers and query text, so
  scrub sensitive fields if that is a concern.
- Do not mix up the ports. gRPC is 4317, HTTP is 4318, and the protocol setting
  has to match the port.

## Common problems

- No traces at all. The endpoint or protocol is wrong, or the app cannot reach
  the collector over the network. Check `docker compose logs otel-collector` and
  confirm the app and collector share a network.
- Traces exist but logs do not link. The trace id is missing from the logs or the
  collector pattern does not match your format. Look at a raw log line and check
  the id is really there.
- A trace stops at a service boundary. A proxy or gateway is dropping the
  `traceparent` header. Allow it through.
- Startup logs have empty trace ids. Expected. Those lines run outside any
  request.
