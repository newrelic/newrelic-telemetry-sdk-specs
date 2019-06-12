# Telemetry SDK specs

The SDK is designed to accept telemetry data send it to New Relic. It's a semantic extension of the public-facing HTTP Telemetry Ingest API, which is a common interface for multiple telemetry backends. It's not an agent or tracer, so it doesn’t instrument anything, but it tries to be helpful (and not too clever). It addresses two primary use cases:

1. You want to record telemetry within an application. For example, you record a summary metric with every request and you’d like to just hand the metric you have to the SDK and let it handle the aggregation, batching, and sending.
 
2. You have a "bag of metrics" or other telemetry that something else created (say, Prometheus or OpenCensus) and you want to get them to New Relic efficiently.

## Specs

* [Communication with New Relic](./communication.md)
* [APIs](./sdk-apis.md)
* [Validation](./validation.md)
* [Configuration & Logging](./configuration-and-logging.md)
* [Limits](./limits.md)
