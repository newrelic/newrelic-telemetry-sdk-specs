# Telemetry SDK specs
The telemetry SDK specs is a documentation repository that describes the basic features shared across language implementations of the New Relic telemetry SDKs. You should use these specs to help you understand what is included in the Telemetry SDK, why it was designed the way it is, and how it works so that you can extend it appropriately.

## The Telemetry SDK
The New Relic Telemetry SDK is designed to accept telemetry data and send it to New Relic. It's a semantic extension of the New Relic public-facing HTTP Telemetry Ingest API, which is a common interface for multiple telemetry backends. It's not an agent or tracer, so it doesn’t instrument anything. It tries to be helpful, so your job of sending telemetry data to New Relic can be done in the right way, easily. Lastly, it is extensible and open source, so that you can tailor it for your use case.

The telemetry SDK's primary use case is: 
 
You have telemetry data, metrics, traces, logs, or events that something else created (say, Prometheus or OpenCensus) and you want to get them to New Relic efficiently. You would use the SDK to quickly write an integration with New Relic. 

You might also do other things with the telemetry SDK. For example, you might want to test integration with New Relic and the telemetry SDK affords you a quick way to get started. Alternatively, you might want to record a simple metric for your application and let the SDK handle aggregation, batching, and sending to New Relic. 

## Specs

* [Communication with New Relic](./communication.md)
* [APIs](./sdk-apis.md)
* [Validation](./validation.md)
* [Configuration & Logging](./configuration-and-logging.md)
* [Limits](./limits.md)