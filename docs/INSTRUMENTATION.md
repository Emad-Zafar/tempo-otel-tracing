# Instrumenting your services for tracing

This guide covers a Spring Boot (Java) service and a Node.js service: how to
emit traces, and how to print the trace id in your logs so the two line up. The
logging setup here is the same one the full observability stack uses to link a
log line to its trace, so doing it now means no rework when you add logs later.

Traces go to Alloy at `alloy:4317` (gRPC) or `alloy:4318` (HTTP). Your service
must be on the `observability-net` Docker network so it can reach Alloy.

## Java and Spring Boot

The goal is two things: JSON logs that carry the trace id, and traces from the
OpenTelemetry Java agent. Neither needs changes to your business code.

### 1. Structured JSON logging with Log4j2

Spring Boot ships Logback by default. Swap it for Log4j2 with the ECS JSON
layout. In your `pom.xml`, exclude the default logging and add Log4j2:

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
  <exclusions>
    <exclusion>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-logging</artifactId>
    </exclusion>
  </exclusions>
</dependency>

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
<dependency>
  <groupId>org.apache.logging.log4j</groupId>
  <artifactId>log4j-layout-template-json</artifactId>
</dependency>
```

If you leave `spring-boot-starter-logging` in, Logback silently wins at runtime.
No error, no warning, just plain text logs where you expected JSON. For a
reactive service, exclude it from `spring-boot-starter-webflux` instead.

Then add `src/main/resources/log4j2-spring.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">
      <JsonTemplateLayout eventTemplateUri="classpath:EcsLayout.json"/>
    </Console>
  </Appenders>
  <Loggers>
    <Root level="INFO">
      <AppenderRef ref="Console"/>
    </Root>
  </Loggers>
</Configuration>
```

Use the name `log4j2-spring.xml`, not `log4j2.xml`, so Spring Boot processes it.
Each log line comes out as one JSON object on stdout, and the agent below fills
in `trace.id` and `span.id`.

### 2. The OpenTelemetry Java agent

Bake the agent into your image and load it at startup:

```dockerfile
ARG OTEL_AGENT_VERSION=v2.12.0
ADD https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/download/${OTEL_AGENT_VERSION}/opentelemetry-javaagent.jar /app/opentelemetry-javaagent.jar

ENTRYPOINT ["sh", "-c", \
  "if [ \"$OTEL_JAVA_AGENT_ENABLED\" = \"true\" ]; then \
     exec java -javaagent:/app/opentelemetry-javaagent.jar $JAVA_OPTS -jar /app/app.jar; \
   else \
     exec java $JAVA_OPTS -jar /app/app.jar; \
   fi"]
```

The conditional entrypoint lets you turn the agent off for one service without
rebuilding. Set the agent up with environment variables on the service:

```yaml
environment:
  OTEL_JAVA_AGENT_ENABLED: "true"
  OTEL_SERVICE_NAME: my-service
  OTEL_EXPORTER_OTLP_ENDPOINT: http://alloy:4317
  OTEL_EXPORTER_OTLP_PROTOCOL: grpc
  OTEL_TRACES_EXPORTER: otlp
  OTEL_METRICS_EXPORTER: none
  OTEL_LOGS_EXPORTER: none
  OTEL_PROPAGATORS: tracecontext,baggage
  OTEL_RESOURCE_ATTRIBUTES: deployment.environment=local
```

Metrics and logs exporters are set to `none` on purpose. Metrics come from
Prometheus scraping your actuator endpoint, and logs come from Log4j2 to stdout
to Loki. The agent only handles traces here.

A traced log line then looks like this, with the ids added for you:

```json
{"@timestamp":"2025-04-20T14:32:01Z","log.level":"INFO","message":"User authenticated","service.name":"my-service","trace.id":"4bf92f3577b34da6a3ce929d0e0e4736","span.id":"00f067aa0ba902b7"}
```

The agent auto-instruments Spring MVC and WebFlux, JDBC, MongoDB, HTTP clients
and more, and it propagates the trace context across service calls, so a request
that crosses services shows up as one trace.

## Node.js

### 1. Structured JSON logging

Log JSON so fields are queryable in Loki. Pino is a good default:

```js
const pino = require("pino");
const logger = pino({ level: process.env.LOG_LEVEL || "info" });

logger.info({ userId: "123", action: "session_start" }, "session started");
```

Winston with `format.json()` works the same way.

### 2. Traces with the OpenTelemetry SDK

Set the OTLP environment variables on the service:

```yaml
environment:
  OTEL_SERVICE_NAME: my-node-service
  OTEL_EXPORTER_OTLP_ENDPOINT: http://alloy:4317
  OTEL_PROPAGATORS: tracecontext,baggage
```

Install the SDK and auto-instrumentation:

```bash
npm install @opentelemetry/sdk-node \
            @opentelemetry/auto-instrumentations-node \
            @opentelemetry/exporter-trace-otlp-grpc
```

Create a `tracing.js` that starts the SDK. It reads the environment variables
above, so there is nothing to hard code:

```js
const { NodeSDK } = require("@opentelemetry/sdk-node");
const { getNodeAutoInstrumentations } = require("@opentelemetry/auto-instrumentations-node");
const { OTLPTraceExporter } = require("@opentelemetry/exporter-trace-otlp-grpc");

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter(),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
```

Load it before your app so it can patch the libraries you use:

```json
{ "scripts": { "start": "node --require ./tracing.js index.js" } }
```

### 3. Trace id in Node logs

Read the active trace id and add it to the log so it matches the derived field
the same way the Java logs do:

```js
const { trace } = require("@opentelemetry/api");

const traceId = trace.getActiveSpan()?.spanContext().traceId;
logger.info({ trace_id: traceId }, "processing request");
```

## Confirm it works

1. Send some traffic through your instrumented service.
2. Open Grafana at `http://localhost:3030` (no login, anonymous admin).
3. Explore, Tempo, search by service name. You should see traces.
4. Open a trace and read the span waterfall to see where time went.
5. Check the Alloy pipeline at `http://localhost:12345`. The
   `otelcol.receiver.otlp` component should be green.

## Do's and don'ts

Do:

- Use one service name per service, the same on every instance.
- Set `deployment.environment` so staging and production do not blur together.
- Keep `trace.id` in your logs so the log to trace link works.
- Let the trace context flow. Do not strip `traceparent` headers at proxies.
- Sample when trace volume is high.

Don't:

- Do not leave `spring-boot-starter-logging` in the Java build, or Logback wins
  and you lose JSON logs.
- Do not put user ids or emails into span attributes or log labels. High
  cardinality blows up storage.
- Do not use a different service name per instance. You lose the aggregate view.
- Do not mix up the ports. gRPC is 4317, HTTP is 4318, and the protocol setting
  has to match the port.
