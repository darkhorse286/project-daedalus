FROM otel/opentelemetry-collector-contrib:0.104.0 AS collector
FROM alpine:3.20
COPY --from=collector /otelcol-contrib /otelcol-contrib
USER nobody
ENTRYPOINT ["/otelcol-contrib"]
