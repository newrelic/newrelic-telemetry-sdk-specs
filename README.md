# Telemetry SDK

The SDK is designed to take telemetry data that already exists and send it to New Relic. It's an extension of the public telemetry ingest HTTP API. It's not an agent or tracer, so it doesn’t instrument anything, but it tries to helpful (and not too clever). It addresses two primary use cases:

1. You want to record telemetry within an application. For example, you record a summary metric with every request and you’d like to just hand the metric you have to the SDK and let it handle the aggregation, batching, and sending.
 
2. You have a "bag of metrics" or other telemetry that something else created (say, Prometheus or OpenCensus) and you want to get them to New Relic efficiently.

The Telemetry SDK suits these needs with two levels of API.

![image](telemetry-sdk-API-levels.png)

## Telemetry API

 The top-level telemetry API offers a harvester that can aggregate, batch, and send the data to New Relic on a regular interval. It handles communication with New Relic and will automatically retry requests, split overly large payloads, and backoff in the face of rate limiting. This level showcases the best practices for sending data to New Relic and is what almost everyone should use.

### Harvester

#### Batching
* The harvester batches telemetry using the configured interval

#### Aggregation
* The harvester handles aggregation for metrics
* Metrics are aggregated differently by type, see below

#### Error handling
* The harvester offers error handling in the face of errors?

## Low-level API

While the telemetry API tries to be helpful, it is supported by a low-level “no frills” API that won’t do anything unless asked explicitly. It knows about the New Relic data structures, like spans and summary metrics. It can accept a batch of telemetry data and return an HTTP request object or send it for you and return the response object. If you really need to directly manipulate and send data, this API offers that full control.

### Batches
* A batch has methods to accept telemetry
* It can return a request object or make the call itself and return the response

### Metrics
* To what extent is this just documenting the metrics backend?

#### Gauge

#### Delta count

#### Cumulative count
* Special logic here

#### Summary

### Spans
* To what extent is this just documenting the traces backend?

### Validation
* What the SDK validates versus what each telemetry backend validates
* NRDB constraints
* NrIntegrationErrors

![image](high-level-MELT-architecture.png)

## Configuration
* API key
* Host url override
* Harvest interval

## Communication with New Relic
* Request format: API keys, User-Agent, other headers
* Common JSON format
* Vortex response codes (public link?) & expected behavior

## Logging
* What should be logged?
